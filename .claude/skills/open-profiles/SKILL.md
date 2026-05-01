# Open Profiles

Open a batch of candidate profiles in the browser. Works with Ashby URLs, LinkedIn URLs, or any web URLs.

## Usage

When the user says "open these profiles", "open them in Chrome", "show me their Ashby pages", or similar:

1. Collect the URLs (from Ashby candidateIds, LinkedIn URLs, or direct URLs)
2. Use the Bash tool to run the `open` command with all URLs quoted and space-separated

## Example

```bash
open "https://app.ashbyhq.com/candidate-searches/new/right-side/candidates/abc123" "https://app.ashbyhq.com/candidate-searches/new/right-side/candidates/def456"
```

## Building URLs

- **Ashby profile URL**: `https://app.ashbyhq.com/candidate-searches/new/right-side/candidates/{candidateId}`
- **LinkedIn**: use the linkedInUrl from the candidate's Ashby profile

## Rules

1. Always quote each URL individually in the `open` command
2. Can open up to ~20 URLs at once without issues
3. If the user asks to "open 6 through 14" from a numbered list, open only those specific candidates
