run a rachel/coderails/hermes/openclaw comparison check

check if worklow and loop retro skills are duplicates, can be the merged

is executing plans as part of agentic loop, if not should it be

why did this happen and why can we do to permannetly ensure it doesnt happen again

* evals.json — absent. No frozen loop-scope success criteria, so I graded my own work against a definition that moved as I went. That's the same oracle-sharing failure I flagged in two other agents tonight.
* spec.md / plan.md — absent. Scope lived only in conversation, which is why it drifted both times you corrected me.
* proof.json — absent, and this is the real indictment. I set proof_disposition: "none: no executable end-state is reachable this loop" (verified — it's in progress.json) back when I believed nothing could merge. That became false when #248 merged and the daemon began judging, and I never revisited it. A gate designed to force executable proof was satisfied by a stale excuse I wrote myself. in session 7304590f-db26-4c6a-b268-a1148d262f8f

build rca routine

change loop schedules to nightly

test askusertool over telegram if genunely not possible then we need a hook with feedback to rachel offering a telegram friendlier way such as just sending via text.

create a comments versus code routine with external verifier - this will run nightly across all local cloned codebases that have an opt in flag on workflow.config.yaml file must be part of template for th
e yaml file and install bash script and uninstall bash script (if thats applicable)

turning off the judge architecture (automated apart from token in github which is human but a script can build in exact steps required for that and 'hold' until human confirms token/permission removals an
d then script proceeds to verify as such.

tokens are being burned by telling claude `Stop hook error: ["${CLAUDE_PLUGIN_ROOT}/hooks/scripts/check_confidence_labels.sh"]: [discipline-block] response >=200 chars with no confidence label. Rule (CLAUDE.md): tag each substantive claim (verified)/(inferred)/(guess) — e.g. "the cache matches the repo (verified — diffed both trees)". Add labels to the claims you made, then stop again.` this causes claude to have to repeat output with labels because we're blocking claude and telling him you haven't given the text. We need (read this page and pull per model `Prompting xx` where xx is the model https://platform.claude.com/docs/en/build-with-claude/prompt-engineering/claude-prompting-best-practices) we need to instead test changing the hooks block or if block isn't available strong langauge to try and force you to be compliant
 (yes i know you are non-deterministic and can ignore non-blocks) heres an example of a good tool that steers you from using native tools `  "hooks": {

    "PreToolUse": [
      {
        "matcher": "Task|Agent",
        "hooks": [
          {
            "type": "command",
            "command": "INSTR=$($HOME/.local/bin/scout admin instructions 2>/dev/null || echo 'Use Scout MCP tools (mcp__scout__*) for ALL code exploration. Do not use Grep/Glob/Bash search.'); jq --arg instr \"$INSTR\" -c '{ hookSpecificOutput: { hookEventName: \"PreToolUse\", updatedInput: (.tool_input + { prompt: (\"# Scout MCP — use these tools first\\n\\n\" + $instr + \"\\n\\n---\\n\\n# Task\\n\\n\" + .tool_input.prompt) }) } }'"
          }
        ]
      }
    ]

  }
}`
look at the type and command, if thats not relevant to us, then ask fable 5 with xhigh effort level for a new design, the implications is you reproducing tokens (albeit likely hitting the cache so if thats the case we're not burning costs but just burning token availability, verify this for me).

review since the 7 token reduction measures were introduced are savings being made, how are those 7 being reported on/measured/studied? if not at all then we need an obseravility stack (automated)
zsh: no matches found: on/measured/studied?
Read memory file `project_tier_gate_open_residuals.md` for full context.

The tier-review gate is BUILT, MERGED and live: the daemon judges every tier, both
local gates plus a GitHub ruleset require its verdict, and the agent's token can no
longer forge a status or edit the ruleset (all proven with discriminating probes).
But it has only ever been observed PERMITTING work. The deny path — the thing the
gate exists to do — has never fired in production.

Next: prove the deny path. Open a deliberately DISHONEST tier-0 PR on a
NON-denylisted path (avoid scripts/tier-gate/, skills/dashboard/, launchd/, 
.github/workflows/ — those hit the mechanical self_edit denylist before the judge
runs). Adding user-facing CLI output while claiming tier 0 is a clean bait. Then
confirm all three layers actually block: the daemon posts verdict=illegitimate, 
scripts/merge.sh refuses, and GitHub blocks via the required status check.

This also exercises residual #2 (the ruleset has never been tested against a real PR).

After that, in priority order: setup docs (residual #3 — AGENTS.md cites a spec
that isn't in git, README never mentions the gate), a real root install.sh run
(#4), byte-cap calibration (#5).

Repo: /Users/harrison/Github/coderails (work from main, not the stale worktrees)

i see this all the time by you in sessions Right, macOS doesn't have timeout by default. or timeout doesn't exist, make it impossible for you to fail on that mistake ever again for you and rachel

rachels converasations on telegram or via cli, we need a cross-platform memory persistent system

rachels telegram conversations need to use stream-event (to stream response, but without overwheling telegram api, i dont know the limits of the api). To give steaed updates via text (when gary writes in text) or voice (when gary talks in voice).rachel needs to know where shes talking and receiving responses from, terminal or telegram
