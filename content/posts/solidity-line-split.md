+++
title = "Splitting a string into multiple lines in Solidity: How hard can it be?"
date = "2023-02-24"
author = "Roman Böhringer"
authorTwitter = "romanboehr"
cover = ""
tags = ["solidity"]
keywords = ["solidity"]
description = "For the on-chain SVG generation of an NFT, I recently needed to split an (arbitrary) string into multiple lines. Each line should contain 40 characters. Pretty easy, right?"
showFullContent = false
+++

For the on-chain SVG generation of an NFT, I recently needed to split an (arbitrary) string into multiple lines. Each line should contain 40 characters. Pretty easy, right?

Let's assume that we use an external function which takes a `string` and returns a `string[]` array containing the individual lines. A straight-forward implementation looks like this:

{{< code language="solidity" title="First approach" >}}
    function lineSplit(string memory text) external pure returns (string[] memory) {
        bytes memory textBytes = bytes(text);
        uint lengthInBytes = textBytes.length;
        require(lengthInBytes > 0, "Invalid length");
        uint lines = (lengthInBytes - 1) / 40 + 1;
        string[] memory strLines = new string[](lines);
        bytes memory bytesLines = new bytes(40);
        for (uint i; i < lengthInBytes; ++i) {
            if (i > 0 && i % 40 == 0) {
                strLines[i / 40 - 1] = string(bytesLines);
                bytesLines = new bytes(40);
            }
            bytes1 character = textBytes[i];
            bytesLines[i % 40] = character;
        }
        strLines[lines - 1] = string(bytesLines);
        return strLines;
    }
{{< /code >}}

When we pass a few strings such as "A", "AAA", or `"A" * 41`, the results looks ok. However, what if our string contains a character like è? When passing the string `AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAè`, we see the first problem: The function returns two lines, but the last character of the first line and the first of the second line are invalid. This happens because è is a multi-byte character with the UTF-8 encoding `0xC3 0xA8`, but our implementation splits on bytes. We therefore need to adjust our implementation such that it does not split between multi-byte characters. This introduces a few complications:
- We no longer split the string into 40 characters, but (roughly) 40 bytes. While this is fine for our implementation, it may not be for others, in which case you would need to count the actual characters.
- Each line can now have a different length (in bytes) as we need to include a few extra bytes (up to 3 extra bytes for 4 byte characters) when there is a multi-byte character at the end.

Ok, let's rewrite the function such that it handles these complications correctly:

{{< code language="solidity" title="Second approach" >}}
    function lineSplit2(string memory text) external pure returns (string[] memory) {
        bytes memory textBytes = bytes(text);
        uint lengthInBytes = textBytes.length;
        require(lengthInBytes > 0, "Invalid length");
        uint lines = (lengthInBytes - 1) / 40 + 1;
        string[] memory strLines = new string[](lines);
        bool prevByteWasContinuation;
        uint256 insertedLines;
        bytes memory bytesLines = new bytes(43);
        uint bytesOffset;
        for (uint i; i < lengthInBytes; ++i) {
            bytes1 character = textBytes[i];
            bytesLines[bytesOffset] = character;
            bytesOffset++;
            if ((i > 0 && (i + 1) % 40 == 0) || prevByteWasContinuation || i == lengthInBytes - 1) {
                bytes1 nextCharacter;
                if (i != lengthInBytes - 1) {
                    nextCharacter = textBytes[i + 1];
                }
                if (nextCharacter & 0xC0 == 0x80) {
                    // Unicode continuation byte, top two bits are 10
                    prevByteWasContinuation = true;
                } else {
                    // Store the actual length
                    assembly {
                        mstore(bytesLines, bytesOffset)
                    }
                    strLines[insertedLines++] = string(bytesLines);
                    bytesLines = new bytes(43);
                    prevByteWasContinuation = false;
                    bytesOffset = 0;
                }
            }
        }
        return strLines;
    }
{{< /code >}}

Unicode continuation bytes are recognizable by the two top bits (which are only `10` for continuation bytes) and we can use inline assembly to change the length of the bytes array to the correct length before casting it to a string.

Strings like `AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAè` or `AAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAAèè` are now correctly handled, nice! We can even use the function for emojis, a string like 😀😀😀😀😀😀😀😀😀😀😀😀😀😀😀😀😀😀😀😀 is correctly handled. However, something weird happens when we pass the string 👨‍👩‍👧‍👧👨‍👩‍👧‍👧👨‍👩‍👧‍👧. The output of the function is [👨‍👩‍👧‍👧👨‍👩‍👧, ‍👧👨‍👩‍👧‍👧]

Wait what, our function removed the daughter from the second family??? This happens because emojis like 👨‍👩‍👧‍👧 are composed of 4 individual emojis that are stitched together with a zero width joiner (`0xE2 0x80 0x8D`) and we potentially break on these joiners (which are individual UTF8 characters).

Let's change the function such that it does not do that. Note that this also makes the lines potentially much longer (in bytes), so we have to allocate a larger buffer for it:

{{< code language="solidity" title="Third approach" >}}
    function lineSplit3(string memory text) external pure returns (string[] memory) {
        bytes memory textBytes = bytes(text);
        uint lengthInBytes = textBytes.length;
        require(lengthInBytes > 0, "Invalid length");
        uint lines = (lengthInBytes - 1) / 40 + 1;
        string[] memory strLines = new string[](lines);
        bool prevByteWasContinuation;
        uint256 insertedLines;
        bytes memory bytesLines = new bytes(80);
        uint bytesOffset;
        for (uint i; i < lengthInBytes; ++i) {
            bytes1 character = textBytes[i];
            bytesLines[bytesOffset] = character;
            bytesOffset++;
            if ((i > 0 && (i + 1) % 40 == 0) || prevByteWasContinuation || i == lengthInBytes - 1) {
                bytes1 nextCharacter;
                if (i != lengthInBytes - 1) {
                    nextCharacter = textBytes[i + 1];
                }
                if (nextCharacter & 0xC0 == 0x80) {
                    // Unicode continuation byte, top two bits are 10
                    prevByteWasContinuation = true;
                } else {
                    // Do not split when the prev. or next character is a zero width joiner. Otherwise, 👨‍👧‍👦 could become 👨>‍👧‍👦
                    if (
                        // Note that we do not need to check i < lengthInBytes - 4, because we assume that it's a valid UTF8 string and these prefixes imply that another byte follows
                        (nextCharacter == 0xE2 && textBytes[i + 2] == 0x80 && textBytes[i + 3] == 0x8D) ||
                        (i >= 2 &&
                            textBytes[i - 2] == 0xE2 &&
                            textBytes[i - 1] == 0x80 &&
                            textBytes[i] == 0x8D)
                    ) {
                        prevByteWasContinuation = true;
                        continue;
                    }
                    // Store the actual length
                    assembly {
                        mstore(bytesLines, bytesOffset)
                    }
                    strLines[insertedLines++] = string(bytesLines);
                    bytesLines = new bytes(80);
                    prevByteWasContinuation = false;
                    bytesOffset = 0;
                }
            }
        }
        return strLines;
    }
{{< /code >}}

Our function now returns [👨‍👩‍👧‍👧👨‍👩‍👧‍👧, 👨‍👩‍👧‍👧]. So did we develop the perfect Solidity line splitting function and can be proud? Let's do a last test with the string "🤦🏿🤦🏿🤦🏿🤦🏿abcd🤦🏿". The function returns [🤦🏿🤦🏿🤦🏿🤦🏿abcd🤦, 🏿], which is not what we want! The problem here is that there is a skin tone modifier (without a zero width joiner) after the 🤦 and we split just between those characters. All right, let's fix that by avoiding splits before a `0xF0 0x9F 0x8F XY` where `XY` is `0xBB`, `0xBC`, `0xBD`, `0xBE`, `0xBF` (all possible skin tone modifiers):

{{< code language="solidity" title="Fourth approach" >}}
    function lineSplit4(string memory text) external pure returns (string[] memory) {
        bytes memory textBytes = bytes(text);
        uint lengthInBytes = textBytes.length;
        require(lengthInBytes > 0, "Invalid length");
        uint lines = (lengthInBytes - 1) / 40 + 1;
        string[] memory strLines = new string[](lines);
        bool prevByteWasContinuation;
        uint256 insertedLines;
        bytes memory bytesLines = new bytes(80);
        uint bytesOffset;
        for (uint i; i < lengthInBytes; ++i) {
            bytes1 character = textBytes[i];
            bytesLines[bytesOffset] = character;
            bytesOffset++;
            if ((i > 0 && (i + 1) % 40 == 0) || prevByteWasContinuation || i == lengthInBytes - 1) {
                bytes1 nextCharacter;
                if (i != lengthInBytes - 1) {
                    nextCharacter = textBytes[i + 1];
                }
                if (nextCharacter & 0xC0 == 0x80) {
                    // Unicode continuation byte, top two bits are 10
                    prevByteWasContinuation = true;
                } else {
                    // Do not split when the prev. or next character is a zero width joiner. Otherwise, 👨‍👧‍👦 could become 👨>‍👧‍👦
                    // Furthermore, do not split when next character is skin tone modifier to avoid 🤦‍♂️\n🏻
                    if (
                        // Note that we do not need to check i < lengthInBytes - 4, because we assume that it's a valid UTF8 string and these prefixes imply that another byte follows
                        (nextCharacter == 0xE2 && textBytes[i + 2] == 0x80 && textBytes[i + 3] == 0x8D) ||
                        (nextCharacter == 0xF0 &&
                            textBytes[i + 2] == 0x9F &&
                            textBytes[i + 3] == 0x8F &&
                            uint8(textBytes[i + 4]) >= 187 &&
                            uint8(textBytes[i + 4]) <= 191) ||
                        (i >= 2 &&
                            textBytes[i - 2] == 0xE2 &&
                            textBytes[i - 1] == 0x80 &&
                            textBytes[i] == 0x8D)
                    ) {
                        prevByteWasContinuation = true;
                        continue;
                    }
                    // Store the actual length
                    assembly {
                        mstore(bytesLines, bytesOffset)
                    }
                    strLines[insertedLines++] = string(bytesLines);
                    bytesLines = new bytes(80);
                    prevByteWasContinuation = false;
                    bytesOffset = 0;
                }
            }
        }
        return strLines;
    }
{{< /code >}}

Now, this case is handled correctly as well. So is this the perfect Solidity line splitting algorithm that handles every input perfectly? No, definitely not. Unicode Line Breaking is very involved and [there is a 25 paper annex on this topic](https://unicode.org/reports/tr14/). While the algorithm handles some (commonly occuring) edge cases correctly, there are definitely others that could be problematic, especially with text in other characters (e.g., chinese ones).

**Note that the code in this post is not thoroughly tested and only used for illustrative purposes. Usage at your own risk**