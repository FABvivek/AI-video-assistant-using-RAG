# AI Video Assistant using RAG

An AI-powered meeting and video analysis assistant. Give it a YouTube URL or a
local audio/video file and it will:

- extract and normalize the audio;
- transcribe the audio in English or translate Hinglish to English;
- generate a short meeting title;
- create a professional summary;
- extract action items, key decisions, and unresolved questions; and
- build a local vector index so you can ask questions about the transcript.

The project has both a Streamlit web interface and an interactive command-line
interface.

## How the application works

The processing pipeline is:

```text
YouTube URL or local file
        |
        v
Audio download/conversion (FFmpeg + pydub)
        |
        v
Mono 16 kHz WAV split into 10-minute chunks
        |
        v
Transcription
  - English: local OpenAI Whisper
  - Hinglish: Sarvam speech-to-text translation API
        |
        v
Mistral LLM
  - title
  - summary
  - action items
  - key decisions
  - open questions
        |
        v
Recursive text splitting + Hugging Face embeddings
        |
        v
Chroma vector database
        |
        v
RAG chat grounded in the transcript
```

### Input and audio processing

`utils/audio_processor.py` detects whether the input is an HTTP(S) URL or a
local path. URLs are downloaded with `yt-dlp`; local files are read with
`pydub`. Both paths are converted to mono, 16 kHz WAV audio and split into
10-minute chunks. Temporary downloads and chunks are written under the
application directory.

FFmpeg is required as a system executable because pydub and yt-dlp use it for
media conversion. Installing only the Python `ffmpeg-python` package is not
enough.

### Transcription

`core/transcriber.py` selects the engine from the language setting:

- `english` loads the Whisper model selected by `WHISPER_MODEL` and performs
  local transcription. The default model is `small`.
- `hinglish` sends 25-second WAV pieces to Sarvam's
  `speech-to-text-translate` endpoint and joins the returned English text.
  The 25-second size stays below Sarvam's 30-second synchronous API limit.

Whisper models are downloaded on first use and cached by Whisper. Sarvam
requires an API key in the environment.

### LLM analysis

`core/summarizer.py` and `core/extractor.py` use the Mistral
`mistral-small-latest` model through LangChain. The transcript is split into
3,000-character sections with 200 characters of overlap for summarization.
The partial summaries are then combined into one final summary.

The extractor makes separate LLM calls for:

1. action items, including task, owner, and deadline;
2. key decisions; and
3. unresolved questions or follow-up topics.

The title generator uses the first 2,000 transcript characters and asks for a
professional title of at most eight words.

### Retrieval-augmented chat

`core/vector_store.py` splits the transcript into 500-character chunks with
50-character overlap, creates `all-MiniLM-L6-v2` CPU embeddings, and persists
them in Chroma under `vector_db/`.

`core/rag_engine.py` retrieves the four most similar chunks for each question.
Those chunks are passed to Mistral in a prompt that instructs it to answer only
from the transcript context. If the answer is not present, the assistant is
instructed to say:

> I could not find this information in the meeting transcript.

## Project layout

```text
AI-video-assistant-using-RAG/
├── README.md
└── AI-Video-Assistant-/
    ├── app.py                    # Streamlit web application
    ├── main.py                   # CLI entry point and reusable pipeline
    ├── Requirements.txt          # Python dependencies
    ├── .gitignore                # Ignores secrets and Python cache files
    ├── core/
    │   ├── extractor.py          # Action items, decisions, questions
    │   ├── rag_engine.py         # Retrieval-augmented question answering
    │   ├── summarizer.py         # Title and summary generation
    │   ├── transcriber.py        # Whisper/Sarvam transcription routing
    │   └── vector_store.py       # Chroma and embedding management
    └── utils/
        └── audio_processor.py    # Download, conversion, and chunking
```

Generated runtime data is intentionally not committed:

- `downloades/` contains downloaded or converted media;
- `*_chunk_*.wav` files contain audio chunks;
- `vector_db/` contains the local Chroma database;
- `__pycache__/` and `*.pyc` contain Python bytecode; and
- `.env` contains local secrets.

## Requirements

- Python 3.10 or newer
- FFmpeg available on `PATH`
- A Mistral API key
- A Sarvam API key when using `hinglish`
- Enough disk space and memory for Whisper and the embedding model

The dependency list is in
[`AI-Video-Assistant-/Requirements.txt`](./AI-Video-Assistant-/Requirements.txt).
Whisper and PyTorch can take significant time and disk space to install.

## Installation

From the repository root on Windows PowerShell:

```powershell
cd .\AI-Video-Assistant-
python -m venv .venv
.\.venv\Scripts\Activate.ps1
python -m pip install --upgrade pip
pip install -r .\Requirements.txt
```

If PowerShell prevents activation, either allow local scripts for your user
(`Set-ExecutionPolicy -Scope CurrentUser RemoteSigned`) or run the commands
using the virtual environment's Python directly:

```powershell
.\.venv\Scripts\python.exe -m pip install -r .\Requirements.txt
```

Install FFmpeg separately and verify it:

```powershell
ffmpeg -version
```

## Environment variables

Create `AI-Video-Assistant-\.env` locally. Never commit this file.

```dotenv
MISTRAL_API_KEY=your_mistral_api_key
SARVAM_API_KEY=your_sarvam_api_key

# Optional values:
WHISPER_MODEL=small
SARVAM_STT_MODEL=saaras:v2.5
```

`MISTRAL_API_KEY` is required for titles, summaries, extraction, and RAG chat.
`SARVAM_API_KEY` is required only when the language is `hinglish`. English
transcription uses Whisper locally and does not call Sarvam.

## Run the Streamlit application

```powershell
cd .\AI-Video-Assistant-
.\.venv\Scripts\Activate.ps1
streamlit run .\app.py
```

Then open the local URL printed by Streamlit, normally
`http://localhost:8501`.

In the sidebar:

1. enter a YouTube URL or a local audio/video path;
2. choose `english` or `hinglish`;
3. select **Analyse**; and
4. wait for the pipeline status indicators to complete.

The results page displays the generated title, summary, full transcript,
action items, key decisions, and open questions. The chat area sends questions
to the RAG chain. **Clear Chat** removes only the visible conversation; it does
not delete the vector database.

## Run the command-line interface

```powershell
cd .\AI-Video-Assistant-
.\.venv\Scripts\Activate.ps1
python .\main.py
```

The CLI asks for the source and language, prints the analysis, and then opens
an interactive chat. Type `exit`, `quit`, or `q` to leave the chat.

The reusable function is:

```python
from main import run_pipeline

result = run_pipeline("https://www.youtube.com/watch?v=...", "english")
print(result["summary"])
```

The returned dictionary contains `title`, `transcript`, `summary`,
`action_items`, `key_decisions`, `open_questions`, and `rag_chain`.

## Important implementation details

- Whisper is loaded lazily and reused within a process.
- Sarvam requests are made one piece at a time and temporary Sarvam WAV files
  are removed after each request.
- Mistral is called independently for the title, summary, and each extraction
  category.
- The RAG prompt deliberately restricts answers to retrieved transcript
  content.
- The Streamlit app stores the current result and chat history in
  `st.session_state`; refreshing the app starts a new in-memory UI session.
- Chroma persists embeddings locally in `vector_db/`, so it may be removed
  when a fresh index is needed.

## Troubleshooting

### `ffmpeg` is not recognized

Install FFmpeg and add its `bin` directory to the Windows `PATH`, then open a
new terminal and retry `ffmpeg -version`.

### API authentication errors

Check that `.env` is in `AI-Video-Assistant-`, the variable names exactly match
the examples, and the keys are valid. Restart Streamlit after changing `.env`.

### Hinglish transcription fails

Confirm `SARVAM_API_KEY` is set and that the audio can be split and exported as
WAV by FFmpeg/pydub. Sarvam's synchronous endpoint accepts pieces of at most
30 seconds; this project sends 25-second pieces.

### Processing is slow or memory-heavy

Use a smaller Whisper model such as `base` or `tiny`, or allow more time for
the first model and embedding downloads. CPU transcription and embedding are
expected to be slower than GPU execution.

### Results appear to contain old transcript data

The Chroma collection uses a fixed local directory and collection name. Stop
the application and remove `vector_db/` before processing a completely
unrelated meeting if an isolated fresh index is required.

## Current limitations

- There is no authentication or multi-user isolation.
- The application processes one source at a time.
- Audio chunks and downloads are not automatically cleaned up.
- The vector store uses a fixed local directory and collection name.
- The UI displays generated text as HTML, so transcripts from untrusted sources
  should be treated carefully.
- PDF/TXT libraries are listed as dependencies, but export controls are not
  currently exposed by the UI or CLI.

## License

No license has been specified yet for this repository.
