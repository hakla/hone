# Hone – Voice-driven text entry and editing

## Purpose and scope

Hone connects existing tools for speech recognition, text processing, and text entry on KDE Wayland. It enables fast dictation without an LLM, on-demand text cleanup, and voice-driven editing of selected text.

Hone is a lightweight, standalone CLI. It does not implement its own speech recognition, audio recording, keyboard monitoring, or clipboard management, and it does not require its own daemon. Global keyboard shortcuts are configured through the desktop environment. The tools it uses can be replaced through configurable commands; Voxtype, Ollama, wl-clipboard, and eitype form the intended integration.

## Usage

Four functions are available independently, each of which can be assigned its own global keyboard shortcut.

| Command | Function | Input | Processing |
| --- | --- | --- | --- |
| `hone dictate --raw` | Raw dictation | Speech | Transcription without an LLM |
| `hone dictate` | Dictation with cleanup | Speech | Transcription and cleanup |
| `hone clip` | Clipboard cleanup | Clipboard text | Cleanup |
| `hone edit` | Command mode | Selected text and spoken instruction | Editing the text according to the instruction |

All four functions are handled by Hone. For speech-based functions, the first invocation starts recording; the second invocation of the same command stops recording and retrieves the completed transcription. Clipboard cleanup does not require recording and runs in a single invocation.

While a recording is in progress, any Hone command stops it. The recording is processed in the mode in which it was started; the command that stopped it is not executed itself.

An empty transcription results in neither text output nor a call to the text processing backend. Hone provides visible feedback while LLM processing is in progress.

### Text output

All four functions paste their result into the active application through the clipboard. Text entry is not simulated character by character. The result is not merely written to stdout or left on the clipboard.

If text is selected at the insertion point, the result replaces that selection. Multiline text is pasted as text rather than as a sequence of individual Enter keystrokes. The original clipboard contents are then restored according to the rules in the "Clipboard" section. When pasting, the target application receives the result, not the previous clipboard contents.

Output is delivered to the insertion point that is active at the time of pasting. Hone does not track windows or text selections. If the original selection is lost during processing, the result is pasted at the insertion point that is active at that time.

Hone uses a globally configurable paste command. Applications with different paste shortcuts, particularly terminals, may require a different setting. Hone does not automatically distinguish between applications.

## Raw dictation

Hone retrieves the completed transcription and pastes it without LLM-based post-processing. The dictation tool does not itself insert the text into the active application. This function does not depend on the availability of the text processing backend.

## Dictation with cleanup

Hone retrieves the completed transcription without the dictation tool inserting it into the active application.

The recognized text is processed using the cleanup prompt. If processing succeeds, only the cleaned-up text is pasted, not the original transcription as well.

If text processing fails, for example because the backend is unreachable or does not respond within the configurable timeout, Hone pastes the unprocessed transcription and displays the error.

## Clipboard cleanup

Hone reads the current text contents of the regular clipboard and uses the same cleanup prompt as dictation with cleanup.

The result is pasted into the active application. The original clipboard contents are preserved through restoration according to the rules in the "Clipboard" section. This mode can also be used for text that did not originate from dictation.

If the clipboard contains no text, the operation is aborted with a notification. If text processing fails, the error is displayed and nothing is pasted.

## Cleanup behavior

Cleanup corrects the input without rewriting its content:

- Spelling, grammar, punctuation, and obvious speech recognition errors are corrected; filler words and recognizable false starts are removed.
- The original language, meaning, and, as far as possible, wording are preserved. Text that is already correct remains unchanged.
- Technical terms, API names, classes, methods, variables, and file paths are recognized as such. When the context is unambiguous, spoken identifiers are converted into conventional developer notation, including PascalCase, camelCase, snake_case, kebab-case, and UPPER_SNAKE_CASE.
- Qualified identifiers retain their separators. When identifiers are ambiguous, no names are invented and no unjustified changes are made.
- The output contains only the cleaned-up text. No explanations, introductory sentences, enclosing quotation marks, or Markdown code blocks that are not part of the intended text are added. Existing formatting that belongs to the text is preserved.

Examples:

| Input | Expected result |
| --- | --- |
| `ähm das müssen wir morgen nochmal prüfen` | `Das müssen wir morgen nochmal prüfen.` |
| `Router Service Punkt on routes recognized` (dictated as a qualified class and method name) | `RouterService.onRoutesRecognized` |
| `Die Methode liefert das Ergebnis als JSON.` | `Die Methode liefert das Ergebnis als JSON.` |

## Command mode

The user selects the text to edit in the active application and starts command mode. Hone captures the current text selection at the start of recording. Previously copied text must not be mistaken for the current selection. If no text is selected, the operation is aborted with a notification, without calling the text processing backend.

The user speaks the desired editing instruction. Once recording is complete, Hone retrieves the transcription as the instruction. The selected text and the spoken instruction are passed separately to the text processing backend. Command mode uses its own prompt to perform the requested text transformation rather than applying cleanup alone.

The text processing backend returns only the requested edited text, without additional explanations or unwanted formatting. Explicitly requested formatting, such as a Markdown code block, is part of the result and is preserved.

Hone pastes the result, replacing the text selection if it is still present. The original clipboard contents are restored both after capturing the selection and after pasting, according to the rules in the "Clipboard" section. If text processing fails, the error is displayed and nothing is pasted; the selected text remains unchanged.

Examples:

| Selected text | Spoken instruction | Expected result |
| --- | --- | --- |
| `Das Feature funktioniert nicht richtig und wir müssen uns das nochmal anschauen.` | "Übersetze das ins Englische und formuliere es als GitHub-Issue-Titel." | `Investigate and fix incorrect feature behavior` |
| `Die Methode funktioniert nicht.` | "Übersetze das ins Englische." | `The method does not work.` |

The examples for cleanup and command mode illustrate the expected behavior; they do not prescribe exact model responses for every possible input.

The spoken instruction concerns text editing. Hone does not execute shell commands generated by the model or perform model-directed actions outside text editing.

## Clipboard

The clipboard is used only temporarily to capture a text selection and paste the result. Its previous contents are restored after these operations, even if processing is aborted or fails. Restoration after pasting must not cause the target application to paste the previous clipboard contents instead of the result.

In particular, restoring text must not leave HTML source code on the clipboard instead of the original plain text.

The intended wl-clipboard integration offers one payload representation per invocation, optionally together with the sensitive hint. Complete restoration of all simultaneously offered MIME representations is therefore not guaranteed; for text, a plain-text representation is preferred.

Results temporarily written to the clipboard by Hone for pasting should not appear in the KDE clipboard history (Klipper). Hone marks these contents as sensitive. This is not guaranteed for text that the active application itself writes to the clipboard when copying a selection in command mode. When originally sensitive clipboard contents are restored, their sensitive designation is preserved. Clipboard history behavior is not guaranteed for other clipboard managers.

## Technical constraints

### Implementation

Hone is implemented in Go.

### Speech recognition

Voxtype handles recording and speech recognition. Its built-in LLM post-processing is disabled.

Hone retrieves the transcription for all speech-based functions through the public Voxtype CLI. Recording is started with file output specific to that recording (`record start --file=<path>`) and stopped with `record stop --wait`. The completed text is retrieved from stdout. The regular Voxtype output mode remains unchanged; no separate Voxtype profile is required.

The commands for starting and stopping recording are configurable. An alternative dictation tool must provide equivalent recording control and allow the completed transcription to be retrieved without inserting text into the active application itself.

### Text processing

Ollama is accessed through an OpenAI-compatible HTTP API. The server URL and model are configured centrally. A local Ollama CLI is not required.

The shared cleanup prompt and the command mode prompt are separately configurable. Text processing is not tied to a particular provider; alternatively, a configurable external command can perform the processing. It receives the text to process and, in command mode, the separate instruction as well. It returns the result as text through stdout.

### Clipboard and keyboard simulation

The commands for reading and writing the clipboard and triggering copy and paste are configurable. The intended integration uses `wl-paste`, `wl-copy`, and `eitype`. eitype is used exclusively to simulate keyboard shortcuts. Hone requires neither access to `/dev/uinput`, membership in the `input` group, nor a privileged input service.

To capture a text selection, Hone uses the approach employed by Murmure: save the clipboard, clear it, trigger the copy command, retrieve the newly copied text, and restore the original clipboard contents.

For output, the result is temporarily written to the clipboard, the paste command is triggered, and the original contents are then restored. The intended mechanism for marking temporary results as sensitive is `wl-copy --sensitive` (the KDE MIME hint `x-kde-passwordManagerHint`), which requires wl-clipboard 2.3.0 or later.

## Project

Hone is maintained in a standalone GitHub repository. This spec is the continuously maintained product contract. The user documentation describes installation, configuration, and usage.
