# Image Support Testing Documentation

**Date:** 2025-11-17
**Feature:** Image pasting support for Claude Sonnet 4.5 Complete
**Status:** Ready for manual testing

## Overview

This document outlines the manual testing procedures required to verify that image pasting works correctly in OpenWebUI after implementing the image format transformation from OpenAI's `image_url` format to Anthropic's `image` format.

## Prerequisites

Before testing, ensure:
1. The modified `function.py` has been uploaded to OpenWebUI (Workspace → Functions)
2. You have access to Docker logs: `docker logs -f open-webui`
3. LOG_LEVEL is set to "DEBUG" in the function valves for detailed logs

## Logging Configuration

The function includes configurable logging through the Valves:

- **LOG_LEVEL**: Set to "DEBUG" (default) for detailed transformation logs
  - Location: `function.py` line 180-183
  - Options: "DEBUG", "INFO", "WARNING", "ERROR"
  - Default: "DEBUG"

When LOG_LEVEL is set to "DEBUG", you should see these log messages during image processing:
- "Transformed image_url to Anthropic image format"
- "Transformed base64 image with media_type=image/png" (or jpeg/gif)
- "Transformed URL image: [url preview]..."

## Test Scenarios

### Test 1: PNG Image (Base64 Encoded)

**Objective:** Verify PNG images can be pasted and processed correctly

**Steps:**
1. Open OpenWebUI chat interface
2. Paste a PNG image directly into the chat input
3. Add text prompt: "What do you see in this image?"
4. Submit the message
5. Monitor Docker logs: `docker logs -f open-webui`

**Expected Results:**
- No `invalid_request_error` about `image_url` format
- Log shows: "Transformed image_url to Anthropic image format"
- Log shows: "Transformed base64 image with media_type=image/png"
- Claude successfully responds with image description
- No API errors in the response

**Test Data Format:**
```
data:image/png;base64,iVBORw0KGgoAAAANSUhEUgA...
```

---

### Test 2: JPEG Image (Base64 Encoded)

**Objective:** Verify JPEG images can be pasted and processed correctly

**Steps:**
1. Open OpenWebUI chat interface
2. Paste a JPEG image directly into the chat input
3. Add text prompt: "Describe this photo"
4. Submit the message
5. Monitor Docker logs

**Expected Results:**
- No `invalid_request_error` about `image_url` format
- Log shows: "Transformed base64 image with media_type=image/jpeg"
- Claude successfully responds with image description

**Test Data Format:**
```
data:image/jpeg;base64,/9j/4AAQSkZJRgABAQEA...
```

---

### Test 3: GIF Image (Base64 Encoded)

**Objective:** Verify GIF images can be pasted and processed correctly

**Steps:**
1. Open OpenWebUI chat interface
2. Paste a GIF image directly into the chat input
3. Add text prompt: "What's in this image?"
4. Submit the message
5. Monitor Docker logs

**Expected Results:**
- No `invalid_request_error` about `image_url` format
- Log shows: "Transformed base64 image with media_type=image/gif"
- Claude successfully responds with image description

**Test Data Format:**
```
data:image/gif;base64,R0lGODlhAQABAIAAAP...
```

---

### Test 4: External Image URL (If Supported)

**Objective:** Verify external HTTP/HTTPS URLs are handled correctly

**Steps:**
1. If OpenWebUI supports external image URLs, provide an image URL
2. Add text prompt: "Analyze this image"
3. Submit the message
4. Monitor Docker logs

**Expected Results:**
- Log shows: "Transformed URL image: [url preview]..."
- Transformed to Anthropic format with `type: "url"`
- Claude successfully processes the image

**Test Data Format:**
```
https://example.com/image.png
```

**Note:** OpenWebUI may not support external URLs by default. If this test fails, it may be a limitation of OpenWebUI, not the transformation code.

---

### Test 5: Malformed Data URL

**Objective:** Verify graceful error handling for malformed image data

**Steps:**
1. Manually construct a malformed data URL (if possible in testing)
2. Submit the message
3. Monitor Docker logs

**Expected Results:**
- Log shows: "Malformed data URL: [error], defaulting to image/jpeg"
- Function doesn't crash
- Attempts to process with fallback media type (image/jpeg)

---

### Test 6: Mixed Content (Text + Image)

**Objective:** Verify that messages with both text and images work correctly

**Steps:**
1. Paste an image into the chat
2. Add a detailed text prompt before or after the image
3. Submit the message
4. Monitor Docker logs

**Expected Results:**
- Both text and image content are processed
- Text appears as `type: "text"` in processed messages
- Image appears as transformed `type: "image"` format
- Claude responds to both the text and image

---

### Test 7: Multiple Images

**Objective:** Verify multiple images can be sent in a single message

**Steps:**
1. Paste multiple images into the chat (if OpenWebUI supports this)
2. Add text prompt: "Compare these images"
3. Submit the message
4. Monitor Docker logs

**Expected Results:**
- Each image is transformed separately
- Multiple "Transformed base64 image" log entries
- Claude processes all images in the message

---

## Verification Checklist

After running tests, verify:

- [ ] No `invalid_request_error` messages about `image_url` format
- [ ] Debug logs show "Transformed image_url to Anthropic image format"
- [ ] Debug logs show correct media types (png/jpeg/gif)
- [ ] Claude successfully describes/analyzes images
- [ ] No API errors or timeout issues
- [ ] Text content still processes correctly in mixed messages
- [ ] Function doesn't crash with malformed data

## Known Limitations

1. **External URLs**: OpenWebUI may not support external image URLs by default
2. **Image Size**: Very large images may exceed Anthropic API limits
3. **Unsupported Formats**: WebP or other formats may not be fully supported by Anthropic API

## Error Messages to Watch For

### Before Fix (Expected Errors):
```
invalid_request_error: messages.1.content.1: Input tag `image_url` found...
```

### After Fix (Should NOT appear):
- The above error should be completely eliminated

### Acceptable Warnings:
```
Malformed data URL: [details], defaulting to image/jpeg
Image content missing URL, passing through as-is
```

## Test Results Template

Use this template to document your test results:

```markdown
## Test Results - [Date]

**Tester:** [Name]
**Environment:** OpenWebUI [version], Docker [version]
**LOG_LEVEL:** DEBUG

### Test 1: PNG Image
- Status: ✓ Pass / ✗ Fail
- Notes: [observations]

### Test 2: JPEG Image
- Status: ✓ Pass / ✗ Fail
- Notes: [observations]

### Test 3: GIF Image
- Status: ✓ Pass / ✗ Fail
- Notes: [observations]

### Test 4: External URL
- Status: ✓ Pass / ✗ Fail / N/A
- Notes: [observations]

### Test 5: Malformed Data
- Status: ✓ Pass / ✗ Fail
- Notes: [observations]

### Test 6: Mixed Content
- Status: ✓ Pass / ✗ Fail
- Notes: [observations]

### Test 7: Multiple Images
- Status: ✓ Pass / ✗ Fail / N/A
- Notes: [observations]

### Overall Assessment
- Image pasting works: Yes / No
- Claude processes images: Yes / No
- Any remaining issues: [describe]
```

## Troubleshooting

### If images still fail:

1. **Check LOG_LEVEL**: Ensure it's set to "DEBUG"
2. **Verify function upload**: Re-upload `function.py` to OpenWebUI
3. **Check Docker logs**: Look for any Python exceptions or API errors
4. **Verify API key**: Ensure Anthropic API key is valid and has proper permissions
5. **Check image format**: Verify the image data URL starts with "data:image/"

### If no logs appear:

1. Verify LOG_LEVEL is "DEBUG" in function valves
2. Check that logger is configured: `logger.setLevel(log_level)` runs at initialization
3. Restart OpenWebUI container if needed

## Next Steps

After completing manual testing:
1. Fill out the test results template above
2. Document any issues encountered
3. Verify all test scenarios pass
4. Proceed to Task 4: Update Documentation (if tests pass)
5. Consider deployment to production environment

## References

- Implementation Plan: `docs/plans/2025-11-17-image-support.md`
- Design Document: `docs/plans/2025-11-17-image-support-design.md`
- Main Code: `function.py` (lines 504+ for image transformation)
