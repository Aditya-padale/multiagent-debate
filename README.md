# Multiagent Debate

Multiagent Debate is a browser-based debate studio. Enter a question and five AI personas argue it through structured rounds, vote on the answer, and let a moderator deliver the final ruling.

The app uses a single-page HTML frontend, a small Flask API, and Groq's Qwen model. It also includes a local preset and fallback mode so the interface remains usable without an API response.

## Features

- AI-generated debates about a user-supplied topic.
- Five debating personas plus a neutral moderator.
- Structured opening statements, rebuttals, cross-fire, and final votes.
- Animated transcript playback with typing indicators and active-speaker highlighting.
- Replay, pause/resume, skip-to-verdict, and 1x/2x playback controls.
- Audience messages that receive replies from two selected agents and the moderator.
- A built-in `How this works` preset that does not require an API call.
- Local fallback scripts when the API is unavailable or returns invalid data.
- Responsive TV debate studio layout with light/dark theme support.
- Reduced-motion support, keyboard-accessible controls, and live status updates.

## Cast

| ID | Name | Role |
|---|---|---|
| `prof` | Aditya Padale | Lead Analyst: evidence-first and data-driven |
| `chaos` | Ronit Swami | Devil's Advocate: challenges assumptions |
| `snack` | Varad Bhandare | Systems Thinker: follows second-order effects |
| `doom` | Person 1 | Risk Analyst: finds failure modes and edge cases |
| `sunny` | Person 2 | Innovation Strategist: proposes practical solutions |
| `judge` | Moderator | Keeps order and delivers the final ruling |
| `you` | You (audience) | Represents audience messages in the transcript |

## Project Files

```text
multiagent debate 2/
├── index.html       # Complete frontend: HTML, CSS, JavaScript, and inline SVG avatars
├── server.py        # Flask server and Groq API proxy
├── api/index.py     # Vercel serverless entrypoint
├── vercel.json      # Vercel routing configuration
├── requirements.txt # Python dependencies
└── .env             # Local environment variables; do not commit this file
```

## Requirements

- Python 3.9 or newer
- pip
- A Groq API key for live AI-generated debates

## Setup

1. Install the dependencies:

   ```bash
   pip install -r requirements.txt
   ```

2. Create a `.env` file in the project directory:

   ```env
   GROQ_API_KEY=your_groq_api_key_here
   PORT=5000
   ```

   `GROQ_API_KEY` is required for live generation. `PORT` is optional and defaults to `5000`.

3. Start the server:

   ```bash
   python server.py
   ```

4. Open [http://localhost:5000](http://localhost:5000) in a browser.

## Deploy to Vercel

1. Import this repository into Vercel.
2. Add `GROQ_API_KEY` under the project Environment Variables.
3. Deploy with the default settings. The `api/index.py` function serves both the frontend and `/api/*` routes.

The local `.env` file is only for development; configure the key in Vercel instead of committing it.

The server serves `index.html` at `/` and exposes the API routes described below. Opening the HTML file directly is not recommended because the live API is designed to run through Flask.

## Using the App

### Start a debate

- Select **How this works** for the built-in demonstration.
- Enter a topic of up to 120 characters and select **Start debate** for a live Groq-generated script.

### Control playback

- **Replay** starts the current script again.
- **Pause** and **Resume** control animated playback.
- **Skip to verdict** jumps to the final vote and ruling.
- **Speed 1x / 2x** changes the word-reveal speed.

### Send an audience message

Enter a message of up to 200 characters in **Heckle the debate** and select **Send message**. The app chooses two debaters and the moderator. It requests replies from the server and falls back to local responses if the request fails.

## Architecture

```text
Browser (index.html)
    │
    ├── POST /api/debate { topic }
    └── POST /api/reply  { text, ids, question }
            │
            ▼
       Flask (server.py)
            │
            ▼
      Groq API / Qwen
```

The frontend is self-contained: its styles, JavaScript, preset debate, fallback responses, and SVG avatar artwork are all in `index.html`. The Flask server validates request sizes, constructs prompts, extracts JSON from model responses, and classifies common errors.

## API Reference

### `GET /`

Serves the frontend from `index.html`.

### `POST /api/debate`

Request:

```json
{
  "topic": "Is cereal soup?"
}
```

The topic is trimmed and limited to 120 characters. A successful response is a debate script containing `question`, two `options`, `rounds`, and a `final` ruling.

The generated script uses these line shapes:

```json
{
  "a": "prof",
  "t": "A spoken line",
  "v": 0
}
```

Stage directions use `{ "s": "A stage direction" }` instead of `a` and `t`. Final votes use `v: 0` or `v: 1` for the selected option.

Possible error responses:

| Status | Error | Meaning |
|---|---|---|
| `400` | `topic is required` | The request has no usable topic |
| `429` | `rate_limited` | Groq rejected the request due to rate limits |
| `502` | `invalid_json` | The model response could not be parsed as JSON |
| `502` | `groq_error` | Another Groq or server error occurred |

### `POST /api/reply`

Request:

```json
{
  "text": "I think snacks should always be involved!",
  "ids": ["prof", "snack", "judge"],
  "question": "Is cereal soup?"
}
```

The server trims `text` to 200 characters, `question` to 160 characters, and accepts at most three IDs from this allowlist:

```text
prof, chaos, snack, doom, sunny, judge
```

A successful response is a JSON array of `{ "a": agent_id, "t": reply }` objects. Each reply is requested as a short, in-character response.

## Configuration

| Variable | Default | Description |
|---|---|---|
| `GROQ_API_KEY` | Required for live AI mode | Server-side Groq authentication |
| `PORT` | `5000` | Flask listening port |

The model is configured in `server.py`:

```python
MODEL = "qwen/qwen3.8-27b"
```

## Fallback Behavior

The frontend can still show the preset debate without Flask or Groq. For custom debates, failed requests use local scripted responses and deterministic topic hashing for vote selection. Audience replies also have local character-specific fallback text.

## Security Notes

- Keep `.env` out of version control.
- The Groq API key is used only by the Flask server and is not sent to the browser.
- Request lengths are capped before prompts are sent to Groq.
- Reply agent IDs are filtered against a fixed allowlist.
- User-provided transcript text is escaped before it is inserted into the page.

## Development

To rename or edit a persona, update the `AG` object and related prompt/fallback text in `index.html`, then update the matching descriptions in `server.py`.

Preset debates live in the `PRESETS` array in `index.html`. Each preset is assembled with the `R(title, lines)` helper. The backend prompt builders are `build_prompt()` and `build_reply_prompt()` in `server.py`.

## Troubleshooting

| Problem | Solution |
|---|---|
| The page does not load | Run `python server.py` and open `http://localhost:5000` |
| Live debate fails | Check `GROQ_API_KEY`, network access, and the Flask terminal output |
| Groq rate-limit message | Wait and retry, or use the built-in preset/fallback mode |
| Generic local responses appear | The server or Groq request was unavailable |
| Port `5000` is busy | Set another value in `.env`, such as `PORT=5050` |

## Dependencies

- Flask
- Flask-CORS
- Groq Python client
- python-dotenv

See `requirements.txt` for the version constraints.
# multiagent-debate
