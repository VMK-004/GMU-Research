# Concepts Notes

Personal notes of programming/research concepts, added one at a time while studying the DCD codebase. Each entry: what it is, how it works, one real example from the code.

---

## argparse (reading command-line arguments in Python)

**What it is:** Python's standard library for reading what you typed in the terminal after the script name.

**The problem it solves:** When you run
`python experiments/dcd/01_create_data.py --config configs/dcd/ioi/gpt2.yaml`,
Python only receives a list of words: `["experiments/dcd/01_create_data.py", "--config", "configs/dcd/ioi/gpt2.yaml"]`. Something has to interpret which words belong together. That is argparse.

**The three standard lines:**

```python
parser = argparse.ArgumentParser()                 # 1. create an empty "form reader"
parser.add_argument("--config", required=True)     # 2. teach it one rule: expect a flag --config, the next word is its value
args = parser.parse_args()                         # 3. actually read the terminal words and store the values in `args`
```

**Key points:**
- The flag name `--config` becomes the attribute `args.config` (argparse strips the `--`).
- `required=True` means the script stops with an error if the flag was not given.
- `args.config` is a plain string, created at runtime from whatever was typed. It is never "defined" as a fixed value in the file.
- Typing a different path after `--config` makes the same script run a different experiment. That is why configs are swapped instead of editing scripts.

**Real example (from `experiments/dcd/01_create_data.py`):**

```python
parser = argparse.ArgumentParser()
parser.add_argument("--config", required=True, help="Path to YAML config file")
args = parser.parse_args()

config = OmegaConf.load(args.config)   # args.config == "configs/dcd/ioi/gpt2.yaml"
```

---

## YAML file (plain-text settings file)

**What it is:** A `.yaml` / `.yml` file that stores labeled values in plain text. It is not a program and does not run by itself.

**Why it exists (in this project):** Experiment settings live in YAML so you can change the experiment by editing/swapping a config file, without rewriting Python scripts.

**Key points:**
- Written as `key: value` pairs.
- Indentation (spaces) shows nesting: a value under a section belongs to that section.
- Lists can be written with `[...]`, e.g. `prompt_types: ["ABBA", "BABA"]`.
- `#` starts a comment (ignored).
- Python reads the file (here via OmegaConf) into an object `config`; then dots follow the nesting, e.g. `config.model.name` → `"gpt2"`.

**Real example (from `configs/dcd/ioi/gpt2.yaml`):**

```yaml
model:
  name: "gpt2"
  cache_dir: "models"

data:
  n_data: 1000
  tasks:
    ioi:
      prompt_types: ["ABBA", "BABA", "filler", "letter", "passive", "3-person"]
```

After `config = OmegaConf.load(...)`:
- `config.model.name` → `"gpt2"`
- `config.data.n_data` → `1000`
