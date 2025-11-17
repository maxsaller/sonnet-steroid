# Image Support Design

## Problem

When users paste images into OpenWebUI, the function fails with an API error:

```
messages.12.content.1: Input tag 'image_url' found using 'type' does not match any of the expected tags: 'document', 'image', ...
```

This occurs because OpenWebUI sends images in OpenAI format (`image_url` type), but Anthropic's API expects a different format (`image` type).

## Solution Overview

Transform image content from OpenAI format to Anthropic format in the message processing pipeline, supporting both pasted images (base64) and URL-referenced images.

## Architecture

### Core Strategy
- Enhance `_process_messages` method to detect and transform `image_url` content types
- Add helper method `_transform_image_content` for clean separation of concerns
- Handle both base64 data URLs and regular HTTP/HTTPS URLs

### Implementation Location
- Modify `_process_messages` method (lines 504-534 in function.py)
- Add new `_transform_image_content` helper method

## Data Format Transformation

### Input Format (OpenAI/OpenWebUI)

**Base64 data URL:**
```json
{
  "type": "image_url",
  "image_url": {
    "url": "data:image/png;base64,iVBORw0KGgoAAAANS..."
  }
}
```

**Regular URL:**
```json
{
  "type": "image_url",
  "image_url": {
    "url": "https://example.com/image.jpg"
  }
}
```

### Output Format (Anthropic)

**For base64:**
```json
{
  "type": "image",
  "source": {
    "type": "base64",
    "media_type": "image/png",
    "data": "iVBORw0KGgoAAAANS..."
  }
}
```

**For regular URLs:**
```json
{
  "type": "image",
  "source": {
    "type": "url",
    "url": "https://example.com/image.jpg"
  }
}
```

## Implementation Details

### New Helper Method: `_transform_image_content`

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
        return {
            "type": "image",
            "source": {
                "type": "url",
                "url": url
            }
        }
```

### Modified `_process_messages` Method

Replace the existing method with enhanced version:

```python
def _process_messages(
    self, messages: List[Dict[str, Any]]
) -> List[Dict[str, Any]]:
    """Process messages to handle different content types"""
    processed = []

    for message in messages:
        content = message.get("content")
        processed_content = []

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
                    logger.debug(f"Transformed image_url to Anthropic format")
                else:
                    # Pass through other types as-is
                    processed_content.append(item)
        else:
            processed_content = [{"type": "text", "text": str(content)}]

        processed.append({
            "role": message["role"],
            "content": processed_content
        })

    return processed
```

## Edge Cases

1. **Malformed data URLs**: Default to `image/jpeg` media type
2. **Missing URL field**: Log warning and pass through original item
3. **Unsupported media types**: Accept any media type from data URL
4. **URLs with query parameters**: Pass through unchanged

## Error Handling

- Non-fatal: Log warnings for malformed content but continue processing
- Graceful degradation: Return original item if transformation fails
- Detailed logging: DEBUG level for successful transforms, WARNING for issues

## Logging Strategy

- **DEBUG**: Log each image transformation with format details
- **WARNING**: Log malformed or unrecognized image formats
- **INFO**: Could add summary of images processed per request (optional)

## Testing Approach

1. Test with pasted PNG images in OpenWebUI
2. Test with pasted JPEG images in OpenWebUI
3. Test with external image URLs (if supported by OpenWebUI)
4. Verify error message no longer appears
5. Verify images display correctly in Claude's response

## Success Criteria

- No more `invalid_request_error` when pasting images
- Images correctly transmitted to Anthropic API
- Both base64 and URL formats supported
- Clean error handling for edge cases
