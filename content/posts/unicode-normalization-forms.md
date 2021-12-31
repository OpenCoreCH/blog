+++
title = "Unicode Normalization Forms: When ö != ö"
date = "2021-06-01"
author = "Roman Böhringer"
authorTwitter = "romanboehr"
cover = ""
tags = ["sre"]
keywords = ["nextcloud", "unicode"]
description = "How special characters in file names can ruin your day."
showFullContent = false
+++

Some time ago, a very weird issue was reported to me about a Nextcloud system. The user uploaded a file with an "ö" on a SMB share that was configured as an external storage in the Nextcloud server. But when accessing the folder containing the file over WebDAV, it did not appear (no matter which WebDAV client was used). After ruling out the usual causes (wrong permissions, etc...), I analyzed the network traffic between the WebDAV client and the server and saw that the file name is indeed not returned after issuing a `PROPFIND`. So I set some breakpoints in the Nextcloud source code to analyze if it is also not returned by the SMB server.
It was returned by the SMB server, but when the Nextcloud system requested more metadata for the file (with the path in the request), the SMB server returned a "file not found" error, which lead Nextcloud to discard the file.
How can it happen that the file is first returned by the SMB server when listing files but then the server suddenly reports an error when requesting more metadata?

When looking at the raw bytes of the first (listing) and second (metadata) SMB request, I found the culprit. The SMB server sent the bytes `0x6F 0xCC 0x88` for the ö, which is [U+006F LATIN SMALL LETTER O](https://www.fileformat.info/info/unicode/char/006f/index.htm) and [U+0308 COMBINING DIAERESIS](https://www.fileformat.info/info/unicode/char/0308/index.htm). In the second request, the Nextcloud server sent `0xC3 0xB6`, which is [LATIN SMALL LETTER O WITH DIAERESIS](https://www.fileformat.info/info/unicode/char/00f6/index.htm). Although the two characters look exactly the same, their code point sequence is different.
This is known as [Unicode equivalence](https://en.wikipedia.org/wiki/Unicode_equivalence) and, in theory, addressed by Unicode normalization. But here, normalization caused this issue. Before storing the file name in the cache, Nextcloud normalized the file name (to NFC) in a function `normalizePath`:

```php
	public static function normalizePath($path, $stripTrailingSlash = true, $isAbsolutePath = false, $keepUnicode = false) {
        ...
		//normalize unicode if possible
		if (!$keepUnicode) {
			$path = \OC_Util::normalizeUnicode($path);
		}
        ...
    }
```
This normalized path is then used for the second request, which fails because the file is stored in a different normalization form (NFD in this case) on the SMB share.

So what can you do about this? Of course you could write a small script to normalize all your file names to NFC on all shares, but this may not be desirable for large shares with millions of files. But it turns out that I was not the first to experience this problem and there is a "NFD compatibility" option that can be set on a per-share basis. When it is enabled, the original [NFD name is also stored](https://github.com/owncloud/core/issues/21365#issuecomment-173637039) in the cache (which results in additional database queries, the option is therefore not enabled per default).