# PROMPTS.md: Living Prompt Pack

> Module 3 · Prompt Chaining. Re-architect the build with prompt chains; capture the reusable ones here.

## How to use this pack

_Each prompt is a reusable step. Chain them: the output of one becomes the input to the next._

## Prompt chain: The review and update flow

### Step 1: Declutter the screen to show opportunity updates only
```
Build the next phase of this app in a strict sequence:
1. Add a screen to update "Opportunities". Match the layout and spacing of the current design.
2. Add a screen "Today" to show the reps their work for today. Match the current content.
3. Navigation: place tabs at the top of the page for Opportunities, Today, and Pipeline.

Build these in order so that Opportunities supports Today. For example, when there are no opportunities, grey out that tab. When there are opportunities, highlight the tab to signal attention.
```

### Step 2: Opportunity updates need Edit flow
```
Apply the following logic constraints to the Opportunities flow:
- Reps can edit the updates. When they save the edits, it also approves the update.
- Reps should also be able to cancel out of the edit, and that puts the update back to pending Approval/Rejection state.
- On Save and Approve failure, trigger the error state: "{{Did not save changes}}" and put the update back to pending Approval/Rejection state..

Maintain the same design language throughout and tether all behavior strictly to these rules.
```

### Step 3: Refine, one surgical polish
```
The opportunities needs a professional color polish.
1. The opportunities layout needs to have the action buttons as the last element.
2. The dark theme is fine, but it should have a light theme too.
3. The gold color is too dingy.
4. Make the reps name header more prominent.

Don't change anything else in the project or touch the underlying logic.
```

## Reusable techniques learned

- It is important to tell the llm what not to touch, just as much as what to touch.
- language matters - how you describe a change can lead to an unexpected outcome.
- It can make changes you don't really notice until you get to edge cases.

## What broke (and the fix)

_Where a single mega-prompt failed and chaining fixed it._

The numbers/metrics on screen kept counting elements that were no longer there.
Instead of showing me all the opportunities, it hid them until the each one is processed.
_____
