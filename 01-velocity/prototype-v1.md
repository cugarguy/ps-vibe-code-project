# Prototype v1: First Build (Lab 1)

> Module 1 · Velocity. Fifteen minutes from a vague problem to a clickable, shareable URL, instinct over methodology.
I first tried using Claude design in our first class and produced this: https://claude.ai/code/artifact/b75f3dac-0b8e-467b-abc0-d479074022bd
I was not able to have Claude or ChatGPT review this link, so I started over using Lovable per the second class. I was able to produce this: https://adapt-and-engage.lovable.app 

## Scenario

_Which of the four scenarios (or your own, instructor-approved) did you build?_
> I chose the CRM scenario because I have an exiting CRM that is overly simplified and not sharable with a team. I can relate to this scenario.

_____

## Launch path

- [x] Copy & Customize (start from a scenario starter prompt)
- [ ] First Screen Method (build only the very first screen the user sees)

## The build

- **What I built:** I built a CRM interface, using the CRM scenario, and applying Faran's "Intro" prompt framework into Claude Design. I had Claude Design interview me on the important elements.
- **Tool used:** Lovable - We chose Claude Design in class due to issue with the Lovable offer not being valid.  by the second class, we were able to use Lovable, so I am including a Lovable link as well 
- **Shareable link:** Claude Design: https://claude.ai/code/artifact/b75f3dac-0b8e-467b-abc0-d479074022bd. and Loavable: https://adapt-and-engage.lovable.app

## Show & Swap read

_What a partner understood from your build with no verbal setup. Their reaction is your first piece of product evidence._
I tired getting ChatGPT and Claude to evaluate as if they were peers. Niether worked with the Claude Design version. I used ChatGPT to review the Lovable version:

- **What they thought it did:** This is a consulting-firm relationship/pipeline system that watches existing work signals—email, calendar, chat, timesheets—and turns them into client context and CRM-like updates without asking consultants to enter data manually. The “Today” view prepares me for client conversations; the “Pipeline” view converts those same signals into an evidence-based forecast. The core promise landed quickly: the system observes the work, proposes what changed, and I confirm rather than maintain CRM records.
- **What surprised them:** I could not tell what the product actually is: CRM replacement, CRM augmentation, personal relationship assistant, or practice-management system. The two screens also imply different primary users—“Today” feels consultant-facing, while “Pipeline” feels partner/practice-lead-facing. Most importantly, I don't know what “Looks right” actually does: does approving “Open a Phase 2 opportunity · $480k” write into Salesforce, change the forecast, create an internal object, or merely train the inference? The confidence percentages look precise but have no meaning I can evaluate, particularly when a 64% inference proposes adding someone to an account team.
- **The gap between what I intended and what they read:** The primary hypothesis appears to be: people who refuse to maintain CRM data will accept a review-and-confirm workflow if the system can infer updates from the digital exhaust of doing their actual jobs. The prototype is therefore testing the human confirmation loop more than “AI can summarize email.” The key behavioral question is whether “Captured for you — confirm or drop” is sufficiently low-friction and trustworthy that users will validate inferred relationship and pipeline state. A second-order hypothesis is that leadership will trust an evidence-graded pipeline more than a manually maintained one, especially if unsupported opportunities are explicitly marked stale rather than silently left in forecast.
