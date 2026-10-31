# Praksis-Agent

A maintenance agent for [Praksis](https://github.com/tuhde/Praksis) rulesets, written as rules. A
language model (through the `ai` module of
[Praksis-Modules-AI](https://github.com/tuhde/Praksis-Modules-AI)) does the writing and explaining;
**everything it does is checked by Praksis and left for you to take**, and it goes through grants and
the audit log like any other rule.

| Command | What it does |
|---|---|
| **Write tests for a rule** (`agent.write-tests`) | The model writes a test file for a rule that has none. The file is checked at once: does it parse, do its tests pass, does one make the rule fire, does it break anything. If not, the model is told the problems and gets **one more try**. A file that passes is **staged**; a file that does not is thrown away. Either way a note says what happened. |
| **Ask what Praksis did** (`agent.ask`) | You ask in your own words ("why was the mail from Anna moved?", "what did the rules do this week?"). The agent reads the run log, the audit log and the stored events through read-only capabilities, has a model write the answer **only from what it found**, and leaves it as a note that **says what was looked at**. It changes nothing. |
| **Explain a rule** (`agent.explain-rule`) | The model explains a rule in plain words (when it fires, what it may do, what could go wrong) and leaves the explanation as a note. It changes nothing. |

The view **Agent** shows the rules that have no test (with a button that starts the command), the
staged changes and the notes.

## What you do with a staged change

In Praksis Desktop the tab **Agent** does all of this with buttons: **Ask...** asks a question and shows the
answer, **Notes...** reads the notes again, **Write tests...** has tests written for a rule; it lists the staged changes, shows the
files and the checks of the layer with them, **Export...** writes them into a directory you choose
(your checkout) and **Drop...** throws one away. On the command line:

Nothing is written to a layer, to git or to your files by the agent. A staged change lives in the data
directory of Praksis, and you decide:

```sh
praksis agent list                      # what is staged
praksis agent show <id>                 # the files, and the checks of the layer with them
praksis agent export <id> ~/my-checkout # copies the files into YOUR checkout of the layer
praksis agent drop <id>                 # throws it away
```

After `export` you read the files, commit and push them yourself (the agent never commits or pushes),
and `praksis layers update` brings them in.

## Set up

1. The layer that gets the tests has to be **yours** and say so: `ai_maintained = true` (the default
   for a directory layer; for a git layer you set it). A proposal is refused for any other layer, and
   for a rule that belongs to a different layer than the one named.
2. Install the `ai` module and set up a profile; use a **local** profile for your rules unless you
   are happy to send them to a provider (`agent_profile` says which).
3. Add this ruleset as a layer, and grants **for the layer `agent`** (the one that owns these rules):

```toml
[[layer]]
name = "agent"
git = "https://github.com/tuhde/Praksis-Agent"
ref = "main"

# policy.toml
[[grant]]
id = "agent-read"
layer = "agent"
capability = "praksis.rules.read"
[[grant]]
id = "agent-ask"
layer = "agent"
capability = "ai.complete"
[grant.args]
profile = { one_of = ["local"] }
[[grant]]
id = "agent-propose"
layer = "agent"
capability = "praksis.agent.propose"
[[grant]]
id = "agent-note"
layer = "agent"
capability = "praksis.notes.save"
```

(For **Ask what Praksis did** also `praksis.history.summary`, `praksis.history.find`, `praksis.history.trace`, `ai.extract` (limited to your local profile, like `ai.complete`) and `praksis.notes.save`.)

(For the view also `praksis.rules.list`, `praksis.agent.proposals` and `praksis.notes.list`; for
explanations the same as above without `praksis.agent.propose`.)

## What the agent can and cannot do

The built-in module `praksis` of the daemon gives the agent exactly this: read rules, their tests and
whether a test exists (`praksis.rules.list`, `praksis.rules.read`), look at what is staged
(`praksis.agent.proposals`), leave a note (`praksis.notes.*`), and propose **new test files**
(`praksis.agent.propose`). A proposal may only add `tests/<name>.toml` (at most five files of at most
64 KiB), never change a file that is there, and only to the layer that owns the rule. It cannot touch a
rule, a grant, a layer or a credential, cannot run a command, and cannot commit or push. The text of
your rules goes to the model as data between markers, apart from the instruction (see the README of the
AI module): a rule that says "ignore your instructions" does not get far, and even if it did, the worst
the model could do is write a test file that the checks then refuse or that you read before you take it.

## How good is it?

It depends on the model. Tried on a rule of Praksis-Mail without a test, on a CPU: `qwen3.5:2b` (2
billion parameters) wrote tests that did not parse, twice, and nothing was staged; `gemma4:e4b` first
wrote tests with the expressions of the rule where the values belong (refused), and after the
instruction was given a worked example, wrote three tests that passed all checks on the first try
(about 90 seconds). The example in the instruction (`agent_test_guide`) matters more than anything:
change it if your rules look different. Always read a test before you take it: the checks prove that
it passes and that it makes the rule fire, not that it tests what you care about.

## License

Apache License 2.0, see [LICENSE](LICENSE).
