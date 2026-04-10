# askQuestions Tool Documentation

## Overview

The `askQuestions` tool enables Copilot to pose clarifying questions to the user during agent-mode conversations. Questions are rendered as an interactive **question carousel** within the chat response stream, allowing the user to answer, skip, or provide free-form text before the agent proceeds.

**Primary Use Cases:**
- Clarifying ambiguous requirements before implementation
- Gathering user preferences on implementation choices
- Confirming decisions that meaningfully affect outcome

**Tool Category:** VS Code Interaction (`ToolCategory.VSCodeInteraction`)

**UI Surface:** Question carousel via `stream.questionCarousel()` (proposed `chatParticipantAdditions` API)

**Feature Gate:** `github.copilot.chat.askQuestions.enabled` (default `true`, tags: `experimental`, `onExp`)

---

## Architecture

### Data Flow

```
LLM generates tool call
        │
        ▼
┌─────────────────────┐
│  prepareInvocation() │  ← validates input, builds confirmation messages
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│   resolveInput()     │  ← captures IBuildPromptContext (provides stream)
└─────────┬───────────┘
          │
          ▼
┌─────────────────────┐
│     invoke()         │
│                     │
│  1. Convert IQuestion[] → ChatQuestion[]  (_convertToChatQuestion)
│  2. stream.questionCarousel(chatQuestions, true)  ← awaits user answers
│  3. stream.progress('Analyzing your answers...')
│  4. Normalize answers  (_convertCarouselAnswers)
│  5. Send telemetry
│  6. Return IAnswerResult as LanguageModelToolResult
└─────────────────────┘
```

### Component Relationships

```
┌──────────────────────────────────────────────────────────────────┐
│  AskQuestionsTool                                                │
│                                                                  │
│  ┌────────────────┐    ┌──────────────────────┐                  │
│  │ resolveInput() │───▶│ _promptContext.stream │                  │
│  └────────────────┘    └──────────┬───────────┘                  │
│                                   │                              │
│  ┌────────────────┐               │                              │
│  │    invoke()    │◀──────────────┘                              │
│  └───────┬────────┘                                              │
│          │                                                       │
│          ├──▶ _convertToChatQuestion()  → ChatQuestion           │
│          │         Maps IQuestion → ChatQuestion                 │
│          │         Determines ChatQuestionType (Text/Single/Multi)│
│          │         Sets defaultValue from recommended options     │
│          │                                                       │
│          ├──▶ stream.questionCarousel()  → Record<string, unknown>│
│          │         Proposed VS Code API                          │
│          │         Returns answers keyed by question id          │
│          │                                                       │
│          └──▶ _convertCarouselAnswers()  → IAnswerResult         │
│                    Normalizes heterogeneous answer formats        │
│                    Handles: string, array, selectedValue,         │
│                    selectedValues, freeformValue, label           │
└──────────────────────────────────────────────────────────────────┘
```

---

## Tool Declaration

Declared in `package.json` under `contributes.languageModelTools`:

```jsonc
{
    "name": "copilot_askQuestions",
    "toolReferenceName": "askQuestions",
    "displayName": "%copilot.tools.askQuestions.name%",          // "Ask Questions"
    "userDescription": "%copilot.tools.askQuestions.description%", // "Ask questions to clarify requirements before proceeding with a task."
    "modelDescription": "Ask the user questions to clarify intent, validate assumptions, or choose between implementation approaches. Prefer proposing a sensible default so users can confirm quickly.\n\nOnly use this tool when the user's answer provides information you cannot determine or reasonably assume yourself. This tool is for gathering information, not for reporting status or problems. If a question has an obvious best answer, take that action instead of asking.\n\nWhen to use:\n- Clarify ambiguous requirements before proceeding\n- Get user preferences on implementation choices\n- Confirm decisions that meaningfully affect outcome\n\nWhen NOT to use:\n- The answer is determinable from code or context\n- Asking for permission to continue or abort\n- Confirming something you can reasonably decide yourself\n- Reporting a problem (instead, attempt to resolve it)\n\nQuestion guidelines:\n- NEVER use `recommended` for quizzes or polls. Recommended options are PRE-SELECTED and visible to users, which would reveal answers\n- Batch related questions into a single call (max 4 questions, 2-6 options each; omit options for free text input)\n- Provide brief context explaining what is being decided and why\n- Only mark an option as `recommended` with a short justification to suggest YOUR preferred implementation choice\n- Keep options mutually exclusive for single-select; use `multiSelect: true` only when choices are additive and phrase the question accordingly\n\nAfter receiving answers:\n- Incorporate decisions and continue without re-asking unless requirements change\n\nAn \"Other\" option is automatically shown to users\u2014do not add your own.",
    "icon": "$(question)",
    "when": "config.github.copilot.chat.askQuestions.enabled",
    "inputSchema": {
        "type": "object",
        "properties": {
            "questions": {
                "type": "array",
                "description": "Array of 1-4 questions to ask the user",
                "minItems": 1,
                "maxItems": 4,
                "items": {
                    "type": "object",
                    "properties": {
                        "header": {
                            "type": "string",
                            "description": "A short label (max 12 chars) displayed as a quick pick header, also used as the unique identifier for the question",
                            "maxLength": 12
                        },
                        "question": {
                            "type": "string",
                            "description": "The complete question text to display"
                        },
                        "multiSelect": {
                            "type": "boolean",
                            "description": "Allow multiple selections",
                            "default": false
                        },
                        "options": {
                            "type": "array",
                            "description": "0-6 options for the user to choose from. If empty or omitted, shows a free text input instead.",
                            "minItems": 0,
                            "maxItems": 6,
                            "items": {
                                "type": "object",
                                "properties": {
                                    "label": {
                                        "type": "string",
                                        "description": "Option label text"
                                    },
                                    "description": {
                                        "type": "string",
                                        "description": "Optional description for the option"
                                    },
                                    "recommended": {
                                        "type": "boolean",
                                        "description": "Mark this option as recommended"
                                    }
                                },
                                "required": ["label"]
                            }
                        },
                        "allowFreeformInput": {
                            "type": "boolean",
                            "description": "When true, allows user to enter free-form text in addition to selecting options. Use when the user's opinion or custom input would be valuable.",
                            "default": false
                        }
                    },
                    "required": ["header", "question"]
                }
            }
        },
        "required": ["questions"]
    }
}
```

Notable schema constraints:
- `questions`: 1–4 items. Headers must be unique (used as answer keys).
- `options`: **optional** (not in `required`). When omitted or empty, the question renders as free-text input (`ChatQuestionType.Text`).
- `options` count: 0 or 2–6. Exactly 1 option is rejected at runtime by `prepareInvocation`.
- `allowFreeformInput`: When `true`, the carousel permits free-text entry alongside option selection.

---

## Input Parameters

### IQuestion

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `header` | `string` | Yes | — | Short label (max 12 chars). Serves as both the display header and the unique key in the answer record. |
| `question` | `string` | Yes | — | Full question text displayed to the user. |
| `multiSelect` | `boolean` | No | `false` | When `true`, the question renders as `ChatQuestionType.MultiSelect` (checkboxes). |
| `options` | `IQuestionOption[]` | No | — | 0 or 2–6 options. Omit for free-text. Exactly 1 option is invalid. |
| `allowFreeformInput` | `boolean` | No | `false` | Permit free-form text entry in addition to predefined options. |

### IQuestionOption

| Property | Type | Required | Default | Description |
|---|---|---|---|---|
| `label` | `string` | Yes | — | Display text and value identifier for the option. |
| `description` | `string` | No | — | Secondary descriptive text. Appended to label as `"label - description"` in the carousel. |
| `recommended` | `boolean` | No | — | Pre-selects the option as default. For single-select, only the first recommended option is used. For multi-select, all recommended options become the default array. |

---

## Output Format

### IAnswerResult

The tool returns a JSON-serialized `IAnswerResult` as a `LanguageModelTextPart` within a `LanguageModelToolResult`.

```typescript
interface IAnswerResult {
    answers: Record<string, IQuestionAnswer>;
}
```

Answers are keyed by `question.header`.

### IQuestionAnswer

| Property | Type | Description |
|---|---|---|
| `selected` | `string[]` | Labels of selected options. Empty array if no option selected. |
| `freeText` | `string \| null` | Free-form text input. `null` when no free text was entered. |
| `skipped` | `boolean` | `true` if the question was unanswered (carousel skipped, no stream, or unknown answer format). |

---

## Implementation

Source: `src/extension/tools/vscode-node/askQuestionsTool.ts`

### Source (editorially formatted; comments condensed for clarity)

```typescript
import * as vscode from 'vscode';
import { ILogService } from '../../../platform/log/common/logService';
import { ITelemetryService } from '../../../platform/telemetry/common/telemetry';
import { CancellationToken } from '../../../util/vs/base/common/cancellation';
import { StopWatch } from '../../../util/vs/base/common/stopwatch';
import { ChatQuestion, ChatQuestionType, LanguageModelTextPart, LanguageModelToolResult, MarkdownString } from '../../../vscodeTypes';
import { IBuildPromptContext } from '../../prompt/common/intents';
import { ToolName } from '../common/toolNames';
import { CopilotToolMode, ICopilotTool, ToolRegistry } from '../common/toolsRegistry';

export interface IQuestionOption {
    label: string;
    description?: string;
    recommended?: boolean;
}

export interface IQuestion {
    header: string;
    question: string;
    multiSelect?: boolean;
    options?: IQuestionOption[];
    allowFreeformInput?: boolean;
}

export interface IAskQuestionsParams {
    questions: IQuestion[];
}

export interface IQuestionAnswer {
    selected: string[];
    freeText: string | null;
    skipped: boolean;
}

export interface IAnswerResult {
    answers: Record<string, IQuestionAnswer>;
}

export class AskQuestionsTool implements ICopilotTool<IAskQuestionsParams> {
    public static readonly toolName = ToolName.AskQuestions;

    private _promptContext: IBuildPromptContext | undefined;

    constructor(
        @ITelemetryService private readonly _telemetryService: ITelemetryService,
        @ILogService private readonly _logService: ILogService,
    ) { }

    async invoke(
        options: vscode.LanguageModelToolInvocationOptions<IAskQuestionsParams>,
        token: CancellationToken
    ): Promise<vscode.LanguageModelToolResult> {
        const stopWatch = StopWatch.create();
        const { questions } = options.input;
        this._logService.trace(`[AskQuestionsTool] Invoking with ${questions.length} question(s)`);

        const stream = this._promptContext?.stream;
        if (!stream) {
            this._logService.warn('[AskQuestionsTool] No stream available, cannot show question carousel');

            // When no stream is available, return a schema-compliant result with all questions marked as skipped
            const skippedAnswers: Record<string, IQuestionAnswer> = {};
            for (const question of questions) {
                skippedAnswers[question.header] = {
                    selected: [],
                    freeText: null,
                    skipped: true
                };
            }

            const toolResultJson = JSON.stringify({ answers: skippedAnswers });
            return new LanguageModelToolResult([
                new LanguageModelTextPart(toolResultJson)
            ]);
        }

        // Convert IQuestion array to ChatQuestion array
        const chatQuestions = questions.map(q => this._convertToChatQuestion(q));
        this._logService.trace(`[AskQuestionsTool] ChatQuestions: ${JSON.stringify(
            chatQuestions.map(q => ({ id: q.id, title: q.title, type: q.type }))
        )}`);

        // Show the question carousel and wait for answers
        const carouselAnswers = await stream.questionCarousel(chatQuestions, true);
        this._logService.trace(`[AskQuestionsTool] Raw carousel answers: ${JSON.stringify(carouselAnswers)}`);

        // Immediately show progress to address the long pause after carousel submission
        stream.progress(vscode.l10n.t('Analyzing your answers...'));

        // Convert carousel answers back to IAnswerResult format
        const result = this._convertCarouselAnswers(questions, carouselAnswers);
        this._logService.trace(`[AskQuestionsTool] Converted result: ${JSON.stringify(result)}`);

        // Calculate telemetry metrics from results
        const answers = Object.values(result.answers);
        const answeredCount = answers.filter(a => !a.skipped).length;
        const skippedCount = answers.filter(a => a.skipped).length;
        const freeTextCount = answers.filter(a => a.freeText !== null).length;
        const recommendedAvailableCount = questions.filter(
            q => q.options?.some(opt => opt.recommended)
        ).length;
        const recommendedSelectedCount = questions.filter(q => {
            const answer = result.answers[q.header];
            const recommendedOption = q.options?.find(opt => opt.recommended);
            return answer && !answer.skipped && recommendedOption
                && answer.selected.includes(recommendedOption.label);
        }).length;

        this._sendTelemetry(
            options.chatRequestId,
            questions.length,
            answeredCount,
            skippedCount,
            freeTextCount,
            recommendedAvailableCount,
            recommendedSelectedCount,
            stopWatch.elapsed()
        );

        const toolResultJson = JSON.stringify(result);
        this._logService.trace(`[AskQuestionsTool] Returning tool result: ${toolResultJson}`);
        return new LanguageModelToolResult([
            new LanguageModelTextPart(toolResultJson)
        ]);
    }

    async resolveInput(
        input: IAskQuestionsParams,
        promptContext: IBuildPromptContext,
        _mode: CopilotToolMode
    ): Promise<IAskQuestionsParams> {
        this._promptContext = promptContext;
        return input;
    }

    private _convertToChatQuestion(question: IQuestion): ChatQuestion {
        // Determine question type based on options and multiSelect
        let type: ChatQuestionType;
        if (!question.options || question.options.length === 0) {
            type = ChatQuestionType.Text;
        } else if (question.multiSelect) {
            type = ChatQuestionType.MultiSelect;
        } else {
            type = ChatQuestionType.SingleSelect;
        }

        // Find default value from recommended option
        let defaultValue: string | string[] | undefined;
        if (question.options) {
            const recommendedOptions = question.options.filter(opt => opt.recommended);
            if (recommendedOptions.length > 0) {
                if (question.multiSelect) {
                    defaultValue = recommendedOptions.map(opt => opt.label);
                } else {
                    defaultValue = recommendedOptions[0].label;
                }
            }
        }

        return new ChatQuestion(
            question.header,       // id
            type,                  // ChatQuestionType
            question.header,       // title
            {
                message: question.question,
                options: question.options?.map(opt => ({
                    id: opt.label,
                    label: `${opt.label}${opt.description ? ' - ' + opt.description : ''}`,
                    value: opt.label
                })),
                defaultValue,
                allowFreeformInput: question.allowFreeformInput ?? false
            }
        );
    }

    protected _convertCarouselAnswers(
        questions: IQuestion[],
        carouselAnswers: Record<string, unknown> | undefined
    ): IAnswerResult {
        const result: IAnswerResult = { answers: {} };

        if (carouselAnswers) {
            this._logService.trace(
                `[AskQuestionsTool] Carousel answer keys: ${Object.keys(carouselAnswers).join(', ')}`
            );
            this._logService.trace(
                `[AskQuestionsTool] Question headers: ${questions.map(q => q.header).join(', ')}`
            );
        }

        for (const question of questions) {
            if (!carouselAnswers) {
                // User skipped all questions
                result.answers[question.header] = { selected: [], freeText: null, skipped: true };
                continue;
            }

            const answer = carouselAnswers[question.header];
            this._logService.trace(
                `[AskQuestionsTool] Processing question "${question.header}", ` +
                `raw answer: ${JSON.stringify(answer)}, type: ${typeof answer}`
            );

            if (answer === undefined) {
                result.answers[question.header] = { selected: [], freeText: null, skipped: true };

            } else if (typeof answer === 'string') {
                // String answer: match against option labels (case-sensitive)
                if (question.options?.some(opt => opt.label === answer)) {
                    result.answers[question.header] = { selected: [answer], freeText: null, skipped: false };
                } else {
                    result.answers[question.header] = { selected: [], freeText: answer, skipped: false };
                }

            } else if (Array.isArray(answer)) {
                // Array answer: coerce elements to strings
                result.answers[question.header] = {
                    selected: answer.map(a => String(a)),
                    freeText: null,
                    skipped: false
                };

            } else if (typeof answer === 'object' && answer !== null) {
                // Object answer: VS Code returns various shapes
                const answerObj = answer as Record<string, unknown>;

                // Extract freeform text (empty string treated as absent)
                const freeformValue =
                    ('freeformValue' in answerObj
                        && typeof answerObj.freeformValue === 'string'
                        && answerObj.freeformValue)
                        ? answerObj.freeformValue
                        : null;

                if ('selectedValues' in answerObj && Array.isArray(answerObj.selectedValues)) {
                    // Multi-select: { selectedValues: string[] }
                    result.answers[question.header] = {
                        selected: answerObj.selectedValues.map(v => String(v)),
                        freeText: freeformValue,
                        skipped: false
                    };
                } else if ('selectedValue' in answerObj) {
                    const value = answerObj.selectedValue;
                    if (typeof value === 'string') {
                        if (question.options?.some(opt => opt.label === value)) {
                            result.answers[question.header] = {
                                selected: [value], freeText: freeformValue, skipped: false
                            };
                        } else {
                            // selectedValue not a known option → treat as free text
                            result.answers[question.header] = {
                                selected: [], freeText: freeformValue ?? value, skipped: false
                            };
                        }
                    } else if (Array.isArray(value)) {
                        result.answers[question.header] = {
                            selected: value.map(v => String(v)),
                            freeText: freeformValue,
                            skipped: false
                        };
                    } else if (value === undefined || value === null) {
                        if (freeformValue) {
                            result.answers[question.header] = {
                                selected: [], freeText: freeformValue, skipped: false
                            };
                        } else {
                            result.answers[question.header] = {
                                selected: [], freeText: null, skipped: true
                            };
                        }
                    }
                } else if ('freeformValue' in answerObj && freeformValue) {
                    // Only freeform text, no selection
                    result.answers[question.header] = {
                        selected: [], freeText: freeformValue, skipped: false
                    };
                } else if ('label' in answerObj && typeof answerObj.label === 'string') {
                    // Raw option object
                    result.answers[question.header] = {
                        selected: [answerObj.label], freeText: null, skipped: false
                    };
                } else {
                    // Unknown object format
                    this._logService.warn(
                        `[AskQuestionsTool] Unknown answer object format for ` +
                        `"${question.header}": ${JSON.stringify(answer)}`
                    );
                    result.answers[question.header] = { selected: [], freeText: null, skipped: true };
                }

            } else {
                // Unknown primitive type (number, boolean, etc.)
                this._logService.warn(
                    `[AskQuestionsTool] Unknown answer format for ` +
                    `"${question.header}": ${typeof answer}`
                );
                result.answers[question.header] = { selected: [], freeText: null, skipped: true };
            }
        }

        return result;
    }

    private _sendTelemetry(
        requestId: string | undefined,
        questionCount: number,
        answeredCount: number,
        skippedCount: number,
        freeTextCount: number,
        recommendedAvailableCount: number,
        recommendedSelectedCount: number,
        duration: number
    ): void {
        this._telemetryService.sendMSFTTelemetryEvent('askQuestionsToolInvoked',
            { requestId },
            {
                questionCount,
                answeredCount,
                skippedCount,
                freeTextCount,
                recommendedAvailableCount,
                recommendedSelectedCount,
                duration,
            }
        );
    }

    prepareInvocation(
        options: vscode.LanguageModelToolInvocationPrepareOptions<IAskQuestionsParams>,
        token: vscode.CancellationToken
    ): vscode.ProviderResult<vscode.PreparedToolInvocation> {
        const { questions } = options.input;

        // Validate input early before showing UI
        if (!questions || questions.length === 0) {
            throw new Error(vscode.l10n.t(
                'No questions provided. The questions array must contain at least one question.'
            ));
        }

        for (const question of questions) {
            // Options with 1 item don't make sense - need 0 (free text) or 2+ (choice)
            if (question.options && question.options.length === 1) {
                throw new Error(vscode.l10n.t(
                    'Question "{0}" must have at least two options, or none for free text input.',
                    question.header
                ));
            }
        }

        const questionCount = questions.length;
        const headers = questions.map(q => q.header).join(', ');
        const message = questionCount === 1
            ? vscode.l10n.t('Asking a question ({0})', headers)
            : vscode.l10n.t('Asking {0} questions ({1})', questionCount, headers);
        const pastMessage = questionCount === 1
            ? vscode.l10n.t('Asked a question ({0})', headers)
            : vscode.l10n.t('Asked {0} questions ({1})', questionCount, headers);

        return {
            invocationMessage: new MarkdownString(message),
            pastTenseMessage: new MarkdownString(pastMessage)
        };
    }
}

ToolRegistry.registerTool(AskQuestionsTool);
```

---

## VS Code Proposed API Types

The carousel UI depends on the proposed `chatParticipantAdditions` API. These types are defined in `src/extension/vscode.proposed.chatParticipantAdditions.d.ts` and re-exported via `src/vscodeTypes.ts`.

### ChatQuestionType

```typescript
export enum ChatQuestionType {
    /** A free-form text input question. */
    Text = 1,
    /** A single-select question with radio buttons. */
    SingleSelect = 2,
    /** A multi-select question with checkboxes. */
    MultiSelect = 3
}
```

### ChatQuestionOption

```typescript
export interface ChatQuestionOption {
    /** Unique identifier for the option. */
    id: string;
    /** The display label for the option. */
    label: string;
    /** The value returned when this option is selected. */
    value: unknown;
}
```

### ChatQuestion

```typescript
export class ChatQuestion {
    /** Unique identifier for the question. */
    id: string;
    /** Text, SingleSelect, or MultiSelect. */
    type: ChatQuestionType;
    /** The title/header of the question. */
    title: string;
    /** Optional detailed message or description. */
    message?: string | MarkdownString;
    /** Options for SingleSelect or MultiSelect questions. */
    options?: ChatQuestionOption[];
    /** Default selected option id(s). */
    defaultValue?: string | string[];
    /** Whether to allow free-form text input alongside options. */
    allowFreeformInput?: boolean;

    constructor(
        id: string,
        type: ChatQuestionType,
        title: string,
        options?: {
            message?: string | MarkdownString;
            options?: ChatQuestionOption[];
            defaultValue?: string | string[];
            allowFreeformInput?: boolean;
        }
    );
}
```

### ChatResponseQuestionCarouselPart

```typescript
export class ChatResponseQuestionCarouselPart {
    /** The questions to display in the carousel. */
    questions: ChatQuestion[];
    /** Whether users can skip answering the questions. */
    allowSkip: boolean;

    constructor(questions: ChatQuestion[], allowSkip?: boolean);
}
```

### ChatResponseStream.questionCarousel

```typescript
interface ChatResponseStream {
    questionCarousel(
        questions: ChatQuestion[],
        allowSkip?: boolean
    ): Thenable<Record<string, unknown> | undefined>;
}
```

Returns `undefined` when the user dismisses the entire carousel. Otherwise, returns a record keyed by question `id` (which is set to `question.header` by the tool). Values are heterogeneous — see the answer normalization section below.

---

## Answer Normalization: `_convertCarouselAnswers`

The carousel API returns answers in various shapes depending on VS Code's internal UI implementation. `_convertCarouselAnswers` normalizes these into a uniform `IQuestionAnswer` structure.

### Dispatch Table

| Raw Answer Shape | Condition | Result |
|---|---|---|
| `undefined` (whole record) | `carouselAnswers === undefined` | All questions → `skipped: true` |
| `undefined` (per-question) | `answer === undefined` | That question → `skipped: true` |
| `string` | Matches an option label (case-sensitive) | `selected: [answer]` |
| `string` | No option match | `freeText: answer` |
| `string[]` (array) | Any elements | `selected: elements.map(String)` |
| `{ selectedValues: string[] }` | Multi-select | `selected: values.map(String)`, optional `freeText` |
| `{ selectedValue: string }` | Matches option label | `selected: [value]`, optional `freeText` |
| `{ selectedValue: string }` | No option match | `freeText: freeformValue ?? value` |
| `{ selectedValue: array }` | Array variant | `selected: value.map(String)`, optional `freeText` |
| `{ selectedValue: null/undefined }` | With `freeformValue` | `freeText: freeformValue` |
| `{ selectedValue: null/undefined }` | Without `freeformValue` | `skipped: true` |
| `{ freeformValue: string }` | Only freeform, no selection keys | `freeText: freeformValue` |
| `{ label: string }` | Raw option object | `selected: [label]` |
| `{ unknownProperty }` | Unrecognized object | `skipped: true` (warning logged) |
| `number`, `boolean`, etc. | Primitive non-string | `skipped: true` (warning logged) |

**Key behaviours:**
- Option matching is **case-sensitive** (`'yes'` does not match `'Yes'`).
- Empty `freeformValue` (`''`) is treated as absent (falsy check); `freeText` is set to `null`.
- When both `selectedValue` and `freeformValue` are present, both are preserved in the result.
- When `selectedValue` does not match any option and `freeformValue` is also provided, `freeformValue` takes precedence as `freeText`.
- Extra keys in `carouselAnswers` not corresponding to any question header are silently ignored.

---

## Supporting Interfaces and Services

### IBuildPromptContext (excerpt)

The `resolveInput` method captures an `IBuildPromptContext` instance to access the `stream` property. Relevant fields:

```typescript
export interface IBuildPromptContext {
    readonly requestId?: string;
    readonly query: string;
    readonly stream?: vscode.ChatResponseStream;   // ← used by askQuestions
    readonly tools?: {
        readonly toolReferences: readonly InternalToolReference[];
        readonly toolInvocationToken: vscode.ChatParticipantToolToken;
        readonly availableTools: readonly vscode.LanguageModelToolInformation[];
    };
    // ... additional fields omitted
}
```

### ICopilotTool<T> (interface)

```typescript
export interface ICopilotTool<T> extends ICopilotToolExtension<T> {
    invoke?: vscode.LanguageModelTool<T>['invoke'];
    prepareInvocation?: vscode.LanguageModelTool<T>['prepareInvocation'];
}
```

The `resolveInput` method comes from `ICopilotToolExtension<T>`:

```typescript
export interface ICopilotToolExtension<T> {
    resolveInput?(input: T, promptContext: IBuildPromptContext, mode: CopilotToolMode): Promise<T>;
    // ... other optional methods
}
```

### CopilotToolMode

```typescript
export enum CopilotToolMode {
    /** Give a shorter result, agent mode can call again to get more context */
    PartialContext,
    /** Give a longer result, it gets one shot */
    FullContext,
}
```

### ToolName

```typescript
// In src/extension/tools/common/toolNames.ts
export enum ToolName {
    AskQuestions = 'ask_questions',  // internal name
    // ...
}

export enum ContributedToolName {
    AskQuestions = 'copilot_askQuestions',  // VS Code registration name
    // ...
}

// Tool category mapping
ToolCategory mapping: ToolName.AskQuestions → ToolCategory.VSCodeInteraction
```

---

## Configuration

### Setting: `github.copilot.chat.askQuestions.enabled`

| Property | Value |
|---|---|
| Type | `boolean` |
| Default | `true` |
| Description | "Allow agent mode to ask clarifying questions before proceeding with a task." |
| Tags | `experimental`, `onExp` |

The `when` clause in the tool declaration (`config.github.copilot.chat.askQuestions.enabled`) ensures the tool is only registered when the setting is enabled.

---

## Telemetry

### Event: `askQuestionsToolInvoked`

Fired once per `invoke()` call, after answers are collected:

| Property | Classification | Type | Description |
|---|---|---|---|
| `requestId` | SystemMetaData | `string \| undefined` | Chat request turn ID (`options.chatRequestId`) |
| `questionCount` | SystemMetaData | measurement | Total questions asked |
| `answeredCount` | SystemMetaData | measurement | Questions answered (not skipped) |
| `skippedCount` | SystemMetaData | measurement | Questions skipped |
| `freeTextCount` | SystemMetaData | measurement | Questions answered with free text |
| `recommendedAvailableCount` | SystemMetaData | measurement | Questions with a `recommended` option |
| `recommendedSelectedCount` | SystemMetaData | measurement | Questions where user chose the recommended option |
| `duration` | SystemMetaData | measurement | Total time in milliseconds (StopWatch elapsed) |

GDPR annotations are present in the source for data governance compliance.

---

## Test Coverage

Test file: `src/extension/tools/vscode-node/test/askQuestionsTool.spec.ts`

Tests exercise `_convertCarouselAnswers` via a `TestableAskQuestionsTool` subclass that exposes the protected method. The test file defines local type aliases rather than importing from the source.

### Test Suites

| Suite | Cases | Key Behaviours Verified |
|---|---|---|
| `carouselAnswers is undefined` | 1 | All questions marked skipped |
| `answer undefined for a question` | 2 | Individual skipped; partial answers handled |
| `answer is a string` | 4 | Option matching, free-text fallback, no-options case, empty string |
| `answer is an array` | 3 | Multi-select, non-string coercion, empty array |
| `selectedValue (VS Code format)` | 4 | Matching option, non-matching → free text, array value, free-text question |
| `selectedValues (VS Code multi-select)` | 3 | Multi-select, empty array, non-string coercion |
| `label property` | 1 | Raw option object format |
| `freeform text with options` | 8 | freeformValue only, null/undefined selectedValue + freeform, both selection + freeform, multi-select + freeform, empty freeformValue, non-matching selectedValue + freeform, skipped on null selectedValue without freeform |
| `unknown format` | 4 | Unknown object, number, boolean, null |
| `edge cases` | 6 | Empty questions array, special chars in headers, unicode headers, mixed formats, case-sensitive matching, extra keys ignored |

Total: **36 test cases** across 10 suites.

---

## Portability Assessment

### API Dependencies

| Dependency | Status | Impact |
|---|---|---|
| `vscode.ChatResponseStream.questionCarousel()` | **Proposed** (`chatParticipantAdditions`) | Cannot be used by marketplace extensions without special enablement. Core dependency for the carousel UI. |
| `ChatQuestion`, `ChatQuestionType`, `ChatQuestionOption` | **Proposed** | Types for constructing carousel questions. |
| `ChatResponseQuestionCarouselPart` | **Proposed** | Part type for the carousel response. |
| `vscode.LanguageModelTool<T>` | Stable | Standard tool interface. |
| `vscode.l10n.t()` | Stable | Localization API. |

**Verdict:** This tool has a hard dependency on the proposed `chatParticipantAdditions` API for its primary UI mechanism. It cannot be ported to a standalone marketplace extension without either:
1. The `chatParticipantAdditions` proposal being finalized and promoted to stable API, or
2. Implementing a fallback UI (e.g., QuickPick-based, as used in the previous version of this tool).

The `resolveInput` → `IBuildPromptContext` → `stream` chain is also internal infrastructure specific to the Copilot Chat extension's tool registry and is not part of the public VS Code extension API.
