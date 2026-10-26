# File Saving API - Explainer

This proposal adds an API for saving streamable, seekable files from within the browser.

## Use Cases

A website may want to stream a large file to the user's computer. For preexisting files hosted on a server, this can be done via an ordinary HTTP request. However, for *client-generated* files (that is, files created client-side via other JavaScript APIs), there is no straightforward way to do so.

As other Web APIs expose more functionality for working with potentially-large files (for instance, the Web Codecs API, which allows for encoding videos), it is likely that the inability to easily save such files will become a bigger issue.

One motivating example is a web application which applies an effect to a video, encodes it, then allows the user to download the resulting file (for instance, [ntsc-rs-web](https://github.com/ntsc-rs/ntsc-rs-web)).

### Existing APIs

The current APIs for saving files are insufficient for streaming large files to disk:

- The most common approach is to construct a `Blob` containing the complete contents of the file, and save it by presenting (or programmatically clicking) an `<a download="[the desired name]" href="[the blob URL]">` element. This is what the [FileSaver.js library](https://github.com/eligrey/filesaver.js/) does, for instance.

  This approach is inefficient for large files, often prohibitively so, since it requires the file's entire contents to be in memory at once.

- It is also possible to register a service worker to serve client-generated data as a streaming HTTP request. This is what the [StreamSaver.js library](https://github.com/jimmywarting/StreamSaver.js) does.

  This approach is highly convoluted and requires some fragile code to work around the limitations of the Service Worker API. For instance, the service worker needs a "dummy page" to intercept. For StreamSaver.js, this defaults to `https://jimmywarting.github.io/StreamSaver.js/mitm.html`; no bets on what will happen to all the sites using StreamSaver.js right now if that URL ever succumbs to "link rot".

  In addition, this approach does not support seeking within the stream. Seeking is necessary for many file formats which need to write metadata at the beginning of the file, but that metadata is not known until the rest of the file has been written (media container formats, for instance).

- It is possible to use the [File System API](https://developer.mozilla.org/en-US/docs/Web/API/File_System_API) to write a file to the [origin private file system](https://developer.mozilla.org/en-US/docs/Web/API/File_System_API/Origin_private_file_system), obtain a blob URL for that OPFS-backed file (via [`getFile`](https://developer.mozilla.org/en-US/docs/Web/API/FileSystemFileHandle/getFile) and [`createObjectURL`](https://developer.mozilla.org/en-US/docs/Web/API/URL/createObjectURL_static)), and then save that URL.

  This approach avoids the hard limitations of the other approaches (in-memory size and seekability), but involves an unnecessary extra file copy. In addition, the OPFS storage quota for a site [can vary based on arbitrary factors outside a site's control](https://developer.mozilla.org/en-US/docs/Web/API/StorageManager/estimate):

  > You may find that the quota varies from origin to origin. This variance is based on factors such as:
  >
  > - How often the user visits
  > - Public site popularity data
  > - User engagement signals like bookmarking, adding to homescreen, or accepting push notifications

- The [File System Access API](https://wicg.github.io/file-system-access/) provides a `showSaveFilePicker` method which implements the proposed functionality. However, along with it comes a much larger API surface which gives websites significantly more access to the user's file system (as the name implies). This has led to significant opposition from [Mozilla](https://github.com/mozilla/standards-positions/issues/154) and [WebKit](https://github.com/WebKit/standards-positions/issues/28), neither of whom are ever likely to implement said API.

  Additionally, this API is in some sense *too* flexible for the use case outlined here (downloading a client-generated file as if it was being downloaded from a server). Because the File System Access API is centered around providing a "native-like" experience, the UX of `showSaveFilePicker` is centered around *accessing* a file rather than *downloading* one:

  - The method must return a real, full-access handle to the *exact* file that the user picks.

  - The file picker's window has an ominous "Warning: this site can see edits you make" header.

  - The file picker does not warn about overwriting an existing file.

  - Files saved via `showSaveFilePicker` do not appear in the browser's download history.

## Goals

- Allow web applications to provide streaming downloads of client-generated files, including those which are too big to fit in memory or which may require seeking as part of the write process.

- Provide a user experience similar to that of downloading a file from a server.

- Allow web applications to specify a *suggested* filename and file type for the downloaded file(s).

- Expose this functionality through existing Web APIs for writing files (in particular, [`FileSystemWritableFileStream`](https://developer.mozilla.org/en-US/docs/Web/API/FileSystemWritableFileStream) and [`FileSystemSyncAccessHandle`](https://developer.mozilla.org/en-US/docs/Web/API/FileSystemSyncAccessHandle)).

- Allow the user to cancel the download process at any point.

## Non-goals

- Give sites direct read-write access to any *existing* files on the user's file system, regardless of their contents.

- Expose the selected name or location of the saved file.

- Provide any guarantees that the file picker's return value corresponds to any particular individual file.

  - This is intended to support the partial-download strategies used by browsers currently (streaming into `.crdownload`/`.part`/etc. and then renaming them once the download has completed).

- Allow the user to *pause* the download process at any point.

  - Allowing downloads to be paused makes sense for preexisting files being served over HTTP, but for client-generated files, it would introduce a variety of edge cases that would all require careful handling.

- Provide an API to download more than one file, or an entire directory, at once.

- Provide an explicit way to "prefetch" a download / start the stream early while the user is still selecting a destination.

  - This could be accomplished using existing web APIs by streaming to the OPFS, and then switching over to the real destination once the user selects one.

- Provide an API for an application to report the approximate "progress" of a download, although such an API should be able to be added later if desired.

## Proposed Solution

The proposed API adds a new method (name to be bikeshedded) to `Window`, which opens a "save file" picker and allows the user to select where to save the file and what to name it. This method then returns a handle which *only* exposes APIs that modify the file's contents, but notably does not expose any of the file's metadata (e.g. name, last-modified date, identity).

This new method exposes options similar to `showSaveFilePicker`, with a few differences.

```ts
function showDownloadFilePicker(options?: DownloadFilePickerOptions): Promise<DownloadFileHandle>;

interface DownloadFilePickerOptions {
    /** These are all the same as `SaveFilePickerOptions`. */
    types?: FilePickerAcceptType[];
    excludeAcceptAllOption?: boolean;
    id?: string;
    suggestedName?: string;

    /** `FileSystemHandle`s are not accepted for `startIn`. */
    startIn?: WellKnownDirectory;
    /**
     * The application *may* specify the final file size. This allows for
     * progress reporting based on the amount written so far. If specified, the
     * download will fail if the file ever exceeds this size or undershoots it
     * when the download is finished.
     *
     * If `finalSize` exceeds some user-agent-defined download size limit, a
     * `QuotaExceededError` (or something) is thrown.
     */
    finalSize?: number;
}

/**
 * Note that this interface does not inherit from `FileSystemHandle` or
 * `FileSystemFileHandle`. This means it is not possible to get the file's
 * `name`, check if it's the same entry as some other handle, or get it as a
 * `File` object.
 *
 * Emits a `success` event when the download finishes, or an `error` event if
 * the download fails.
 */
interface DownloadFileHandle extends EventTarget {
    readonly status: "pending" | "in-progress" | "succeeded" | "failed";

    /**
     * Note that this does not take any `FileSystemCreateWritableOptions`. This
     * is because those options are not applicable:
     * - `keepExistingData` doesn't matter because no existing data will be preserved
     *   anyway.
     * - `mode` is always treated as "exclusive".
     *
     * In addition to a `NoModificationAllowedError` if another stream or access
     * handle is active, this throws an `InvalidStateError` if another stream or
     * access handle to this file has already been *closed* (which means the
     * download is already "completed"), or the user cancelled the download.
     */
    createWritable(): Promise<FileSystemWritableFileStream>;

    /**
     * In web workers only. There is also no `options` argument; `mode` is
     * treated as `readwrite`.
     *
     * Throws an `InvalidStateError` under the same circumstances as
     * `createWritable`.
     */
    createSyncAccessHandle(): Promise<FileSystemSyncAccessHandle>;
}
```

### Behavior / intended implementation details

While the API surfaces look similar, `showDownloadFilePicker` is intended to be implemented quite differently from `showSaveFilePicker`, in a way that does not require general file system access. As an example which includes the intended implementation *and* UX surface:

- After `showDownloadFilePicker` is called, a file picker is opened. The user selects where they want to save the file, and the browser can choose how to handle cases where e.g. the selected path overwrites an existing file.

- An in-progress download appears in the browser's download manager, shown in a "waiting"/"pending" state, and the `DownloadFileHandle`'s `status` is `"pending"`.

- At the same time, the browser creates a temporary file (`.crdownload`/`.part`/etc), which it then returns a handle to.

  - This is why `DownloadFileHandle.createWritable()` always operates in "exclusive" mode; in effect, the temporary file has already been created by the time it is called.

  - I believe this should also make it safe to return a `FileSystemSyncAccessHandle` outside the OPFS; in essence, the temporary file may not be in the OPFS but is already "isolated" by virtue of being a temporary file.

- The application creates either a `FileSystemWritableFileStream` or `FileSystemSyncAccessHandle`. The in-progress download's visual progress updates to match. The `DownloadFileHandle`'s `status` is `"in-progress"`.

  - If the application did not specify a `finalSize`, the download will show an indeterminate loading animation (as is done for downloads with `Transfer-Encoding: chunked`, HTTP/2 downloads with no `Content-Length`, or downloads with `Content-Encoding: gzip` where `Content-Length` doesn't tell you the downloaded file's actual size).

- The application then writes into the file. The user-visible "download progress" updates based on the total size of the file (so e.g. seeks do not increase it).

  - Any write which brings the file's total size above `finalSize` will throw a `QuotaExceededError` (or similar).

- The application closes the stream or access handle.

- If there was some issue finalizing the download (e.g. a file has ended up being placed where the partial download would have been moved to), or if the file's final size is less than the specified `finalSize`, then the `DownloadFileHandle`'s `status` is `"failed"` and it emits an `error` event.

- Otherwise, the `DownloadFileHandle`'s `status` is `"succeeded"` and it emits a `success` event.

## Alternatives Considered

- **Just returning a `Promise<FileSystemWritableFileStream>` directly from `showDownloadFilePicker`.** This would permanently tie the entire "save file" API to one specific interface for writing files (`FileSystemWritableFileStream`). Even if `FileSystemSyncAccessHandle` support in particular isn't important, it demonstrates that there is more than one way to write a file, and more may be desired in the future.

  Additionally, this would not provide any way to signal when a download succeeds or fails, which may happen asynchronously some time after the stream or access handle is closed.

- **Don't support `FileSystemSyncAccessHandle`.** While it's a bit of a niche use case, it's beneficial to be able to use *existing* file-writing APIs, without artificial limitations.

- **Reworking the File System Access API.** It has been over 7 years since Mozilla rejected the File System Access API, and it is unlikely that any vendor other than Chrome will ever ship it in *any* form. For instance, see [this comment](https://github.com/mozilla/standards-positions/issues/154#issuecomment-863349817):

  > It's unfortunate that all these capabilities are bundled together as it's not clear if we could support only some of them in Firefox at this point or if applications expect all-or-nothing.

  Or [this comment](https://github.com/mozilla/standards-positions/issues/738#issuecomment-1401367367):

  > The idea that there might be elements of the specification that are safe to use is not one that we've explored fully. That said, we try to avoid taking specifications à la carte, because that makes it harder to manage interoperability, especially when it results in more complicated feature detection.

  It's pretty clear that shipping even a subset of the File System Access API will not happen, which is why this proposal is intentionally completely separate (even though it uses some ideas from their interfaces).

  Additionally, the File System Access API is a poor fit for the "download-flavored" UX which this proposal leans towards, as mentioned earlier.

- **Allowing the user to pause downloads.** It's unclear how to surface this behavior. For `FileSystemWritableFileStream`, all the write methods *could* just wait until the user resumes the download, but I seriously doubt that any web application or library expects (or is prepared to deal with) those methods waiting an arbitrarily long time before resolving. In other words, pausable downloads would introduce a new and surprising behavior to an existing interface. This also wouldn't work well with `FileSystemSyncAccessHandle`.

## Privacy & Security Considerations

This API intentionally does not provide any information about the destination file. The intent is for the file handle to be conceptually "isolated" in the same way as the OPFS is, even if it lives on disk as e.g. a `.part` file in the destination folder.

### Resource exhaustion / filling up a user's disk

There is no new risk here compared to existing HTTP downloads. As mentioned above, there are already a number of ways in which an HTTP download can have an indeterminate length, and browsers can use existing mechanisms to avoid filling up the user's disk space.

### Timing attacks

It may theoretically be possible to determine the storage medium to which the user has saved the file by measuring seek and write timings. This is already possible with the OPFS, but may be more of an issue here, since the destination directory could be anywhere while the OPFS typically always lives on one particular disk.

### Malicious downloads

Malicious files may be served, just as with HTTP downloads; this is not a new risk. This API does not preclude the ability to scan the saved file(s) for malware. This can be done after the web application closes the file handle but before a success/error event is emitted.

### (Lack of) persistent read/write access

Unlike `showSaveFilePicker`, the file's download state and lifetime is heavily managed:

- When the file is writable, it is shown to the user as an in-progress download. During this period, other applications are unlikely (see below) to read or write the file, especially since all modifications will end up in a `.crdownload`/`.part` file.

- Once the handle is closed, the download is *complete*. This status is clearly signaled to the user, and the web application cannot further read or write to the saved file.

Since download progress is part of the browser UI and not the web application's UI, and partial downloads are typically kept separate from completed downloads on the filesystem, a web application cannot present a fake "download complete" UI and then sneakily modify the file afterwards.

A web application also cannot read back any modifications later made to the file by other applications, since there is no way to re-open a `DownloadFileHandle`.

### Concurrent modification

While browsers have existing mechanisms to prevent two downloads with the same destination, it is in general not possible to prevent other applications from reading or writing over in-progress downloads. Browsers' existing practice of storing partial downloads as temp files and moving them into place later will prevent other applications from *accidentally* accessing an in-progress download.

### Secret communication

If some external application chooses to read or modify a `.crdownload`/`.part` partial-download file *on purpose,* it is theoretically possible for that external application to communicate with a web application through that channel.

Other methods already exist for external applications to communicate with a cooperating web application (for instance, opening a local web server); however, they are subject to restrictions such as a CSP. Communication via a partially-downloaded file would not be.

The scope of this channel, and the ability to use it for nefarious purposes, is quite limited. It would only have an impact if:

- There is some local application which can read and modify the user's files, but cannot circumvent *any* sandboxing measures from a browser running at the same privilege level.

- There is a compromised web application or website which cannot otherwise communicate with a cooperating local application (for instance, because of a CSP).

- The compromised web application can convince the user to save a file somewhere, allowing the local application to detect it and communicate with the web application.

I don't believe this is an issue under any reasonable security model, but am mentioning it for completeness' sake.

## Unresolved Questions

- Should an application be able to mark a download as "failed" / abort it programmatically? I believe that `FileSystemWritableFileStream.abort()` could do this, but `FileSystemSyncAccessHandle` has no analogous API.

- Is it a problem for the API surface to look so similar to `showSaveFilePicker` and `FileSystemFileHandle`, despite functioning substantially differently?

- What error types should be exposed on `DownloadFileHandle`? I'm not sure if too many different errors would expose too much information about the underlying environment.

- Where and when do we surface `AbortError`s if the user cancels a download? To my surprise, none of the `WritableStream` or `FileSystemWritableFileStream` methods make any mention of what should happen if the underlying `WritableStream` is `abort()`ed.

- Is there any existing mechanism to fire an event when a `WritableStream` or `FileSystemSyncAccessHandle` is `close()`d, as would be necessary for emitting the final success/error event?

- Is it desirable to tie this API to existing File System API interfaces, or are the two likely to evolve in different (conflicting) directions? That is, are `FileSystemWritableFileStream` or `FileSystemSyncAccessHandle` likely to grow any APIs which require "direct" file system access, or is that confined to higher-level "metadata" APIs like `FileSystemHandle`?
