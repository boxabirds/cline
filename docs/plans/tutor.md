# Plan: Integrating Plain English Explanations (Tutoring Feature)

**Overall Status**: in_progress

## Phase 1: Foundational Setup - Instructing the AI

### 1.1. Modify System Prompt (`src/core/prompts/system.ts`)
   - **Status**: completed
   - **Goal**: Instruct the AI on *how* to format and provide educational explanations.
   - **File**: `/Users/julian/expts/cline/src/core/prompts/system.ts`
   - **Action**: Update the `EDUCATIONAL NARRATIVE REQUIREMENTS` section within the `SYSTEM_PROMPT` function.
   - **Details**:
     - Explicitly instruct the AI to use one of the following markers for its plain English explanations:
       1. Prefix the entire explanation with `ED: ` (This marker will allow for multi-line explanations, which are considered to end just before an `ACTION:` tag or at the end of the AI's current message segment).
       2. Enclose the entire explanation within `<explanation>` and `</explanation>` tags.
     - The AI should be told that these explanations must directly precede the technical action or tool usage they describe.
   - **Example addition to `SYSTEM_PROMPT` (within `EDUCATIONAL NARRATIVE REQUIREMENTS`):**
     ```
     1. For each technical action:
        - Provide a plain-English explanation before execution. This explanation MUST be formatted in one of two ways:
          a) Prefix the entire explanation with "ED: " (e.g., "ED: I will now read the file to understand its contents."). The explanation continues until an "ACTION:" tag or the end of your current message.
          b) Enclose the entire explanation within "<explanation>" and "</explanation>" tags (e.g., "<explanation>I am about to list the directory contents to see what files are present.</explanation>").
        - Ensure the explanation is concise and immediately precedes the technical step or tool invocation.
     ```
   - **Considerations**: Ensure UK English spelling (e.g., "plain-English explanation," "behaviour," "customise").

## Phase 2: Core Logic Implementation (`src/core/task/index.ts`)

### 2.1. Implement `extractEducationalContent` Method
   - **Status**: proposed
   - **Goal**: Reliably parse and extract the formatted educational explanation from the AI's raw response string.
   - **File**: `/Users/julian/expts/cline/src/core/task/index.ts`
   - **Method Signature**: `private extractEducationalContent(response: string): string | undefined`
   - **Placement**: Add as a private method within the `Task` class.
   - **Details**:
     - Use a regular expression to detect and capture content marked with `ED: ` or enclosed in `<explanation>...</explanation>` tags.
       - Regex should be: `/(?:ED:\s*(.*?)(?=\s*ACTION:|$)|<explanation>\s*(.*?)\s*<\/explanation>)/s`
       - The `s` flag ensures `.` matches newlines, crucial for multi-line `ED:` explanations.
     - The method should return the first valid captured explanation (either from `ED:` or `<explanation>`).
     - Trim whitespace from the extracted content.
     - Return `undefined` if no valid explanation marker is found.
     - Adhere to UK English spelling in any comments (e.g., "analyse response").

### 2.2. Implement `presentExplanation` Method
   - **Status**: proposed
   - **Goal**: Display the extracted educational explanation to the user in a distinct manner.
   - **File**: `/Users/julian/expts/cline/src/core/task/index.ts`
   - **Method Signature**: `private async presentExplanation(content: string): Promise<void>`
   - **Placement**: Add as a private method within the `Task` class, near `extractEducationalContent`.
   - **Details**:
     - Utilise the existing `this.say()` method.
     - Pass a new, specific `type` for the message, e.g., `"educational"` (or `"explanation_message"`). This allows the UI to style it differently.
     - Provide the extracted `content` as the message.
     - Ensure the `isMarkdown` parameter of `this.say()` is set to `true` to render the explanation with markdown formatting.
     - Example call: `await this.say("educational", content, undefined, true /* isMarkdown */);`

### 2.3. Integrate into `presentAssistantMessage` Method
   - **Status**: proposed
   - **Goal**: Process incoming assistant messages to find, display, and then remove educational explanations before standard content handling.
   - **File**: `/Users/julian/expts/cline/src/core/task/index.ts`
   - **Method**: `async presentAssistantMessage()`
   - **Details**:
     - Locate the section where `block.type === "text"` and `block.content` is processed.
     - **Before** any other processing of `block.content`:
       1. Call `const explanation = this.extractEducationalContent(block.content);`
       2. If `explanation` is truthy:
          a. Call `await this.presentExplanation(explanation);`
          b. Modify `block.content` to remove the explanation text. Use `String.prototype.replace()` with the same regex patterns used for extraction, replacing the match with an empty string. Trim `block.content` afterwards.
          ```typescript
          // Example of cleaning block.content:
          if (explanation) {
              await this.presentExplanation(explanation);
              block.content = block.content
                  .replace(/(?:ED:\s*(.*?)(?=\s*ACTION:|$)|<explanation>\s*(.*?)\s*<\/explanation>)/s, '')
                  .trim();
          }
          ```
     - This ensures explanations are handled first, then removed.

## Phase 3: User Interface (Responsibility of the USER/UI Developer)

### 3.1. Handle New Message Type
   - **Status**: proposed
   - **Goal**: Visually differentiate educational explanations in the chat interface.
   - **Files**: Relevant UI component files.
   - **Action**: The UI code needs to be updated to recognise and style the new message type (e.g., `"educational"`).
   - **Details**:
     - Implement specific CSS styling for messages of type `"educational"`.
     - This could involve a different background colour, a distinct icon, etc.

## Phase 4: Testing and Refinement

### 4.1. Unit Testing
   - **Status**: proposed
   - **`extractEducationalContent`**:
     - Test with `ED:` prefixed content (single and multi-line).
     - Test with `<explanation>` tagged content.
     - Test with content containing `ACTION:` tags after `ED:`.
     - Test with responses lacking any explanation markers.
     - Test edge cases.
   - **`presentExplanation`**: (Focus on `this.say` being called correctly).

### 4.2. Integration Testing (within `src/core/task/index.ts`)
   - **Status**: proposed
   - Verify that `presentAssistantMessage` correctly calls `extractEducationalContent` and `presentExplanation`.
   - Confirm that `block.content` is properly cleaned.
   - Observe `this.say` invocations.

### 4.3. End-to-End Testing
   - **Status**: proposed
   - Craft AI interactions that should trigger educational explanations.
   - Observe the UI for correct display, styling, and content cleaning.
   - Ensure regular AI messages are unaffected.

## Implementation Sequence (Overall Status: in_progress):
1.  **Modify `SYSTEM_PROMPT`** (`src/core/prompts/system.ts`) - **Status**: completed
2.  **Implement `extractEducationalContent`** (`src/core/task/index.ts`) - **Status**: proposed
3.  **Implement `presentExplanation`** (`src/core/task/index.ts`) - **Status**: proposed
4.  **Integrate into `presentAssistantMessage`** (`src/core/task/index.ts`) – extraction, presentation, and cleaning logic - **Status**: proposed
5.  **(USER/UI Team)** Update UI components to handle and style the new `"educational"` message type - **Status**: proposed
6.  Conduct thorough testing across all levels (unit, integration, end-to-end) - **Status**: proposed
7.  Refine based on testing outcomes - **Status**: proposed
