# Image Support Implementation Plan

> **For Claude:** REQUIRED SUB-SKILL: Use superpowers:executing-plans to implement this plan task-by-task.

**Goal:** Fix image pasting in OpenWebUI by transforming OpenAI-format `image_url` content to Anthropic's `image` format.

**Architecture:** Add `_transform_image_content` helper method to handle format conversion, then modify `_process_messages` to detect and transform image content. Support both base64 data URLs and regular HTTP/HTTPS URLs.

**Tech Stack:** Python, Anthropic API, OpenWebUI function framework

---

## Task 1: Add Image Transformation Helper Method

**Files:**
- Modify: `function.py:504` (add new method before `_process_messages`)

**Step 1: Add the `_transform_image_content` helper method**

Add this new method immediately before the `_process_messages` method (around line 504):

```python
def _transform_image_content(self, item: Dict[str, Any]) -> Dict[str, Any]:
    """
    Transform OpenAI image_url format to Anthropic image format.

    Handles:
    - Base64 data URLs (data:image/png;base64,...)
    - Regular HTTP/HTTPS URLs

    Args:
        item: Content item with type="image_url"

    Returns:
        Transformed content item with type="image"
    """
    # Extract URL from nested structure
    image_url_obj = item.get("image_url", {})
    url = image_url_obj.get("url", "")

    if not url:
        logger.warning("Image content missing URL, passing through as-is")
        return item

    # Check if it's a data URL (base64)
    if url.startswith("data:"):
        # Parse: data:image/png;base64,iVBORw0K...
        try:
            # Split into parts
            header, data = url.split(",", 1)
            # Extract media type (e.g., "image/png")
            media_type = header.split(";")[0].split(":")[1]

            logger.debug(f"Transformed base64 image with media_type={media_type}")
            return {
                "type": "image",
                "source": {
                    "type": "base64",
                    "media_type": media_type,
                    "data": data
                }
            }
        except (ValueError, IndexError) as e:
            logger.warning(f"Malformed data URL: {e}, defaulting to image/jpeg")
            # Fallback: try to extract just the data part
            data = url.split(",", 1)[-1] if "," in url else ""
            return {
                "type": "image",
                "source": {
                    "type": "base64",
                    "media_type": "image/jpeg",
                    "data": data
                }
            }
    else:
        # Regular URL
        logger.debug(f"Transformed URL image: {url[:50]}...")
        return {
            "type": "image",
            "source": {
                "type": "url",
                "url": url
            }
        }
```

**Step 2: Verify syntax**

Run: `python -m py_compile function.py`
Expected: No output (successful compilation)

**Step 3: Commit the helper method**

```bash
git add function.py
git commit -m "Add image content transformation helper method"
```

---

## Task 2: Integrate Image Transformation into Message Processing

**Files:**
- Modify: `function.py:504-534` (update `_process_messages` method)

**Step 1: Update the `_process_messages` method to handle image_url**

Find the `_process_messages` method and replace the content processing loop. Locate this section:

```python
if isinstance(content, list):
    for item in content:
        item_type = item.get("type")
        if item_type == "text":
            processed_content.append({
                "type": "text",
                "text": item.get("text", "")
            })
        # Add image/document processing here if needed
        else:
            # Pass through other types as-is
            processed_content.append(item)
```

Replace it with:

```python
if isinstance(content, list):
    for item in content:
        item_type = item.get("type")
        if item_type == "text":
            processed_content.append({
                "type": "text",
                "text": item.get("text", "")
            })
        elif item_type == "image_url":
            # Transform OpenAI format to Anthropic format
            transformed = self._transform_image_content(item)
            processed_content.append(transformed)
            logger.debug("Transformed image_url to Anthropic image format")
        else:
            # Pass through other types as-is
            processed_content.append(item)
```

**Step 2: Verify syntax**

Run: `python -m py_compile function.py`
Expected: No output (successful compilation)

**Step 3: Visual inspection**

Read the modified `_process_messages` method to ensure:
- Text content still processed correctly
- Image URL content now transformed
- Other content types passed through unchanged
- Logic flow is clear and correct

**Step 4: Commit the integration**

```bash
git add function.py
git commit -m "Integrate image transformation into message processing"
```

---

## Task 3: Manual Testing and Verification

**Files:**
- None (testing only)

**Step 1: Check logging configuration**

Verify that DEBUG logging is enabled by checking the valves configuration:
- LOG_LEVEL should be set to "DEBUG" for detailed transformation logs

**Step 2: Test with OpenWebUI**

Manual testing steps:
1. Upload the modified `function.py` to OpenWebUI (Workspace → Functions)
2. Paste a PNG image into the chat
3. Check Docker logs: `docker logs -f open-webui`
4. Verify:
   - No "invalid_request_error" about image_url
   - Log shows "Transformed image_url to Anthropic image format"
   - Log shows "Transformed base64 image with media_type=image/png"
   - Claude responds successfully

**Step 3: Test with different image formats**

Test scenarios:
- PNG image (data:image/png;base64,...)
- JPEG image (data:image/jpeg;base64,...)
- GIF image (data:image/gif;base64,...)
- External URL (if OpenWebUI supports it)

**Step 4: Document test results**

Create: `docs/plans/2025-11-17-image-support-test-results.md`

Document:
- Which formats were tested
- Whether transformation succeeded
- Any error messages encountered
- Whether Claude processed images correctly

**Step 5: Commit test documentation**

```bash
git add docs/plans/2025-11-17-image-support-test-results.md
git commit -m "Add image support test results documentation"
```

---

## Task 4: Update Documentation

**Files:**
- Modify: `README.md` (add image support to features)
- Modify: `CHANGELOG.md` (document the fix)

**Step 1: Update README.md features section**

Find the features section (around line 9-16) and ensure image support is mentioned or implied. The current features already include general Claude capabilities, so add a note if needed:

After the existing features, consider adding:
```markdown
- **🖼️ Image Support** - Paste images directly into chat for multimodal interactions
```

**Step 2: Update CHANGELOG.md**

Add a new entry at the top:

```markdown
## [v4.1.0] - 2025-11-17

### Fixed
- Image pasting now works correctly - transforms OpenAI `image_url` format to Anthropic `image` format
- Support for both base64-encoded images and external image URLs
- Graceful error handling for malformed image data

### Added
- `_transform_image_content` helper method for image format conversion
- Debug logging for image transformation details
```

**Step 3: Verify documentation changes**

Run: `grep -i "image" README.md CHANGELOG.md`
Expected: Should show the new documentation entries

**Step 4: Commit documentation updates**

```bash
git add README.md CHANGELOG.md
git commit -m "Update documentation for image support"
```

---

## Task 5: Final Review and Cleanup

**Files:**
- Review: All modified files

**Step 1: Review code changes**

Check the complete `function.py` for:
- No syntax errors: `python -m py_compile function.py`
- Consistent indentation (4 spaces)
- Proper logging statements
- Clear comments
- No debug print statements left behind

**Step 2: Review commit history**

Run: `git log --oneline -5`
Expected output should show:
- Update documentation for image support
- Add image support test results documentation
- Integrate image transformation into message processing
- Add image content transformation helper method

**Step 3: Check for uncommitted changes**

Run: `git status`
Expected: "nothing to commit, working tree clean"

**Step 4: Review file structure**

Run: `ls -la function.py docs/plans/`
Verify all expected files exist

**Step 5: Final verification**

Read through the implementation once more:
1. `_transform_image_content` handles both base64 and URLs
2. `_process_messages` calls the transformation for image_url type
3. Error handling is graceful (logs warnings, doesn't crash)
4. Documentation is updated

---

## Completion Checklist

- [ ] `_transform_image_content` method added
- [ ] `_process_messages` updated to handle image_url
- [ ] Code compiles without syntax errors
- [ ] Manual testing completed with PNG/JPEG images
- [ ] Test results documented
- [ ] README.md updated
- [ ] CHANGELOG.md updated
- [ ] All changes committed
- [ ] No uncommitted changes remain

## Next Steps

After implementation:
1. Use superpowers:finishing-a-development-branch to merge/PR
2. Deploy updated function.py to OpenWebUI
3. Verify in production with real image paste tests
