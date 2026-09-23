# PCS FreeSWITCH Survey Integration

Lua dialplan script that connects FreeSWITCH to the PCS survey service. It retrieves the active survey for a dial number, plays the survey prompts to a caller, collects DTMF answers, optionally collects an NPS score, and submits the resulting feedback back to the PCS API.

> **Status:** This repository is an integration script rather than a standalone FreeSWITCH module or application. The script is tailored to the PCS API and the deployment paths used by the original environment, so review and configure it before using it in production.

## Features

- Fetches survey metadata from the PCS `getSurveyDetails` endpoint.
- Supports voice surveys with:
  - Rating questions (`1–5` or `0–9`, depending on the PCS rating-range setting).
  - Boolean questions (`1` or `2`).
  - Multiple questions driven by the API response.
- Plays welcome, question, and goodbye audio prompts from the FreeSWITCH sounds directory.
- Reads the connected agent identifier from the `cc_agent` FreeSWITCH channel variable.
- Collects an optional NPS response when the API returns an NPS question.
- Posts question IDs, answers, survey metadata, and callback information to the PCS `updateFeedback` endpoint.
- Hangs up gracefully when no active survey is available.

## How the script works

1. `pcs.lua` requests survey details from PCS using the configured username, password, and dial number.
2. The response is decoded as JSON and used to determine the survey type, question count, prompts, rating range, and audio files.
3. For voice surveys, FreeSWITCH plays each prompt and collects one-digit DTMF answers with `session:playAndGetDigits`.
4. If an NPS question is present, the script collects an additional score.
5. The script plays the goodbye prompt and submits the collected data as JSON to PCS.
6. If no survey is returned, the script plays the goodbye prompt and terminates the call.

## Requirements

- A running FreeSWITCH installation with Lua scripting enabled.
- LuaSocket (`socket.http` and `ltn12`).
- A JSON library available to the FreeSWITCH Lua runtime:
  - `json` for decoding the survey response.
  - `dkjson` for encoding the feedback request.
- Network access from FreeSWITCH to the PCS HTTP API.
- A PCS API deployment exposing:
  - `GET /PCS/survey/getSurveyDetails`
  - `POST /PCS/survey/updateFeedback`
- Survey audio files installed under the FreeSWITCH sounds directory.
- A dialplan or application entry that invokes this script with a live FreeSWITCH `session`.

## Installation

1. Copy `pcs.lua` to a location readable by the FreeSWITCH process, commonly the FreeSWITCH scripts directory.
2. Install the required Lua dependencies in the Lua environment used by FreeSWITCH.
3. Make sure every prompt returned by PCS exists under the configured sounds directory.
4. Update the PCS URL, credentials, dial number, caller/agent data, and audio path in `pcs.lua` for your environment.
5. Invoke the script from the appropriate FreeSWITCH dialplan or call-control flow.
6. Reload the dialplan or restart/reload FreeSWITCH as required by your deployment.

Example dialplan invocation:

```xml
<action application="lua" data="pcs.lua"/>
```

The exact dialplan context and placement depend on how the call is routed in your FreeSWITCH installation.

## Configuration points

The current script keeps several deployment-specific values inline. Review at least these values before deployment:

- PCS API host and port used by `getSurveyDetails` and `updateFeedback`.
- PCS username and password.
- Survey dial number (`dn`).
- FreeSWITCH audio directory.
- Caller ANI, callback ID, agent ID, and agent name sent in the feedback payload.
- The `cc_agent` channel variable used to identify the agent.

For production use, move credentials and environment-specific values to protected configuration or FreeSWITCH channel variables instead of storing them in source code.

## API payloads

### Survey details response

The script expects the PCS response to provide fields such as:

- `surveyId`, `surveyName`, `serviceId`, and `serviceDn`
- `questionsType`, `questionsCount`, `ratingRange`, and `surveyType`
- `welcomePrompt` and `goodbyePrompt`
- `question<N>prompt`, `question<N>Type`, and `question<N>Id`
- Optional `npsQuestionId` and NPS prompt data

### Feedback request

The feedback request includes survey and service identifiers, call/agent metadata, a timestamp, question IDs, answers, and optional NPS data. The current implementation has explicit fields for up to nine questions; adjust the payload if PCS surveys can contain more questions.

## Security notes

- Do **not** commit real PCS credentials, customer data, or production API endpoints to a public repository.
- The current implementation contains credentials and call metadata in the Lua source. Rotate any credentials that have been exposed and replace them with secure configuration.
- The API calls currently use plain HTTP. Use HTTPS and validate certificates whenever the PCS service supports it.
- Avoid logging full caller information, credentials, or survey responses in production logs.
- Validate and constrain values returned by PCS before using them to build local audio paths.

## Known limitations and areas to review

- The script assumes a voice channel when processing the survey; the SMS branch is not implemented.
- Survey and feedback errors are not handled comprehensively. Add HTTP status checks, timeout handling, and retry/backoff behavior before relying on the integration operationally.
- Some values are global Lua variables rather than local variables, which can cause state leakage when scripts share a Lua runtime.
- Prompt and API field naming is tightly coupled to the PCS response format.
- The current question loop and retry logic should be tested with rating, boolean, mixed, empty, and malformed responses.
- The feedback payload uses fixed question fields instead of constructing them dynamically.
- The script expects the FreeSWITCH `session` object and related channel variables to be available.

## Troubleshooting

- **No survey is played:** verify the PCS URL, credentials, dial number, network connectivity, and the response body returned by `getSurveyDetails`.
- **Audio cannot be found:** confirm that the prompt filenames returned by PCS exist below the FreeSWITCH sounds directory and that the FreeSWITCH process can read them.
- **DTMF answers are rejected:** verify the question type and PCS rating range; rating and boolean questions accept different values.
- **Feedback is not recorded:** inspect the FreeSWITCH console for the HTTP response and verify the JSON field names expected by `updateFeedback`.
- **Agent information is missing:** set the `cc_agent` channel variable before invoking the script.

## Repository layout

```text
.
├── pcs.lua     # FreeSWITCH Lua survey flow and PCS API integration
└── README.md   # Setup, behavior, configuration, and operational notes
```

## Contributing

When changing the integration, test against a non-production PCS service and a FreeSWITCH test call. Keep credentials and customer data out of commits, document changes to the PCS response or request schema, and include the affected FreeSWITCH/Lua runtime assumptions in the pull request.

## License

No license is currently specified. Unless a license is added, others should not assume that the code may be copied, modified, or redistributed.
