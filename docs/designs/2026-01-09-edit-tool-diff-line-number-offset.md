# Edit Tool Diff Line Number Offset

**Date:** 2026-01-09

## Context

The edit tool's diff viewer was displaying line numbers starting from 1, regardless of where the actual edit occurred in the file. When editing content at line 50 of a file, the diff would show lines 1, 2, 3... instead of 50, 51, 52... This made it difficult for users to correlate the displayed diff with the actual file location.

## Discussion

**Problem Analysis:**
- The `DiffViewer` component receives only `old_string` and `new_string` from the edit tool
- It generates a diff between these two isolated strings, which naturally starts at line 1
- The actual position in the file where the edit occurs is lost in the display

**Key Questions:**
1. Where should the starting line number be calculated? 
   - Decision: In `applyEdit.ts` since it already processes the file and finds the match position
2. How to handle `replace_all` edits with multiple occurrences?
   - Decision: Show the line number of the first occurrence only

**Data Flow:**
- `applyEdits()` → returns `startLineNumber`
- `edit.ts` → passes to `returnDisplay`
- `Messages.tsx` → extracts and passes to `DiffViewer`
- `DiffViewer` → offsets all rendered line numbers

## Approach

Calculate the starting line number at the point where the match is found in `applyStringReplace`, then propagate it through the return chain to the UI layer where it offsets the displayed line numbers.

## Architecture

### Changes to `src/utils/applyEdit.ts`

1. Modified `applyStringReplace` to return both the result string and the match index:
```typescript
function applyStringReplace(...): { result: string; matchIndex: number }
```

2. Updated `applyEdits` to:
   - Track `firstMatchLineNumber` initialized to 1
   - Convert match index to line number by counting newlines before the match
   - Return `startLineNumber` in the result object

### Changes to `src/tools/edit.ts`

Added `startLineNumber` to the `returnDisplay` object:
```typescript
returnDisplay: {
  type: 'diff_viewer',
  filePath: relativeFilePath,
  originalContent: { inputKey: 'old_string' },
  newContent: { inputKey: 'new_string' },
  absoluteFilePath: fullFilePath,
  startLineNumber,  // new field
}
```

### Changes to `src/ui/DiffViewer.tsx`

1. Added `startLineNumber?: number` to `DiffProps` interface
2. Updated `RenderDiffContent` to accept `startLineNumber` parameter (default: 1)
3. Applied offset to line number display:
```typescript
const lineOffset = startLineNumber - 1;
gutterNumStr = line.newLine ? (line.newLine + lineOffset).toString() : '';
```

### Changes to `src/ui/Messages.tsx`

Extracted `startLineNumber` from `result.returnDisplay` and passed to `DiffViewer` component.
