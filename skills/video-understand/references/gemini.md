# Gemini Video Understanding

Google's Gemini models provide native video understanding with both visual and audio analysis.

## Setup

```bash
pip install google-genai
```

### Option 1: API Key (Google AI Studio)

```bash
export GEMINI_API_KEY="your-api-key"
# OR
export GOOGLE_API_KEY="your-api-key"
```

Get API key: https://aistudio.google.com/apikey

### Option 2: Service Account / ADC (Vertex AI)

```bash
export GOOGLE_APPLICATION_CREDENTIALS="/path/to/service-account.json"
export GOOGLE_CLOUD_PROJECT="your-project-id"
export GOOGLE_CLOUD_LOCATION="us-central1"  # optional, defaults to us-central1
```

Or use Application Default Credentials via `gcloud auth application-default login`.

When using Vertex AI, specify `--provider vertex` or the skill auto-selects it based on available credentials.

## Models

| Model | Best For |
|-------|----------|
| `gemini-3-flash-preview` | Latest, fast (default) |
| `gemini-3-pro-preview` | Highest quality |
| `gemini-2.5-flash` | Stable production fallback |

**Note:** Gemini 3 models are the latest. Use 2.5 for stable production if needed.

## YouTube URL Processing

Gemini can process YouTube URLs directly without downloading:

```python
from google import genai
from google.genai import types
import os

client = genai.Client(api_key=os.environ["GEMINI_API_KEY"])

response = client.models.generate_content(
    model="gemini-3-flash-preview",
    contents=[
        types.Content(
            parts=[
                types.Part.from_uri(
                    file_uri="https://www.youtube.com/watch?v=VIDEO_ID",
                    mime_type="video/mp4",
                ),
                types.Part.from_text(
                    text="Transcribe this video with timestamps. Note any on-screen text."
                ),
            ]
        )
    ],
)

print(response.text)
```

## Local File Processing

```python
from google import genai
from google.genai import types
import os
import time

client = genai.Client(api_key=os.environ["GEMINI_API_KEY"])

# Upload video
video_file = client.files.upload(file="video.mp4")

# Wait for processing
while video_file.state.value == "PROCESSING":
    time.sleep(2)
    video_file = client.files.get(name=video_file.name)

if video_file.state.value == "FAILED":
    raise RuntimeError("Processing failed")

# Generate content
response = client.models.generate_content(
    model="gemini-3-flash-preview",
    contents=[
        types.Content(
            parts=[
                types.Part.from_uri(
                    file_uri=video_file.uri, mime_type=video_file.mime_type
                ),
                types.Part.from_text(
                    text="Analyze this video: describe the scenes, transcribe speech, note any text on screen."
                ),
            ]
        )
    ],
)

print(response.text)
```

## Effective Prompts

### Full Analysis
```
Analyze this video comprehensively:
1. Transcribe all spoken content with timestamps [MM:SS]
2. Describe key visual scenes and transitions
3. Note any on-screen text, titles, or graphics
4. Identify speakers if multiple people appear
5. Summarize the main topics covered
```

### Timestamp-Focused
```
Provide a timestamped transcript in this format:
[00:00] Speaker/Description: Content
[00:15] Speaker/Description: Content
...
Include both spoken words and significant visual events.
```

### Q&A Style
```
Watch this video and answer:
1. What is the main topic?
2. What key points are made?
3. Are there any product mentions or demonstrations?
4. What action items or conclusions are presented?
```

## Limits

- Max video length: ~1 hour
- Supported formats: MP4, MOV, MPEG, AVI, FLV, MKV, WEBM
- Max file size: 2GB via upload API
- Processing time: Usually 30 seconds to 2 minutes

## Error Handling

```python
try:
    response = model.generate_content([prompt, video_file])
except google.api_core.exceptions.InvalidArgument as e:
    if "video is too long" in str(e):
        # Split video or use different approach
        pass
except google.api_core.exceptions.ResourceExhausted as e:
    # Rate limit - wait and retry
    time.sleep(60)
```

## Structured Output

Request JSON output for easier parsing:

```python
response = model.generate_content([
    """Analyze this video and return JSON:
    {
        "transcript": [{"time": "MM:SS", "speaker": "...", "text": "..."}],
        "scenes": [{"time": "MM:SS", "description": "..."}],
        "on_screen_text": [{"time": "MM:SS", "text": "..."}],
        "summary": "..."
    }""",
    video_file
])
```
