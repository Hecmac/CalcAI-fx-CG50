# Calc AI for the Casio fx-CG50
# WIP!

**Calc AI is a tiny language model that runs locally on a Casio fx-CG50 graphing calculator.** Ask it a question it either answers from internal knowledge or formats the job to hand to a dedicated tool for calculations, unit conversions, statistics, graphs and the periodic table. The model has 2.27 million parameters running at Q4, and it generates about six tokens per second at stock clocks.

```
what's 9x10                  ->  Sure, let me work that out.        9*10 = 90
plot x^3 - x^2               ->  Here is the graph.                 [graph card, F6 for full screen]
convert 5 miles to km        ->  Here is the conversion.            5 mi = 8.04672 km
look up gallium              ->  Here is its entry.                 [element card, F6 for details]
why is the sky blue          ->  Air scatters blue light from the sun more than other colours, so the sky looks blue.
who won the 1998 world cup   ->  I'm not sure about that one.
```

Arithmatic is inefficient to train in small models, so the model produces a short, plain text preamble and a symbolic action such as `<calc|9*10>` or `<plot|x^3-x^2>`, a deterministic tool evaluates it and produces the requested output. The action is hidden from the user, the result appears in the reply.

## Installation

1. Download the newest `release/CALCAI*.g3a`.
2. Connect the fx-CG50 by USB and select **USB Flash**.
3. Copy the `.g3a` file to the root of the calculator drive.
4. Eject the calculator. **Calc AI** will appear in the main menu.

Once again, to clarify, **the weights are inside the add-in**, nothing else is needed.

## Controls

| Key | Action |
| --- | --- |
| ALPHA | toggles between letters (`abc`) and numbers (`123`) |
| EXE | Send message |
| Left / Right | Move the cursor (Up / Down jump to the start / end) |
| DEL / AC | Delete left of the cursor / clear the message |
| F1 - F3 | Accept one of the word suggestions shown above the keys |
| F6 | Details for the last answer, various actions |
| OPTN | Open the settings screen |
| EXIT / MENU | Close the add-in |

Punctuation: `.` is a space in letter mode, EXP is `?`, VARS is `,`, in number mode `->` types `=`.

### Settings screen (OPTN)

| Key | Action |
| --- | --- |
| F1 / OPTN | Back to chat |
| F2 | Benchmark + Verification against reference outputs. saves `CAIRTnnn.csv` to the local storage |
| F3 | Overclock sweep benchmark |
| F4 | Toggle the clock between **Default** and **Overclock**. The overclock lasts only while the add-in is open and is reset on exit |
| EXIT / MENU | Close the add-in |

Overclocking results in around 1.6x generation speed, It runs outside Casio's specification, use at your own risk.

## Style/Scope

| Area | Examples |
| --- | --- |
| Calculations | `9x10`, `2pi`, `3(4+5)`, spoken numbers, percentages, word problems, powers, roots, trigonometry (degrees), factorials |
| Statistics | mean, median, sum, min, max, range, standard deviation of a list |
| Conversions | length, mass, time, volume, temperature (common names and abbreviations) |
| Graphs | any expression in `x`: sums, products, quotients, powers, `sin`, `cos`, `sqrt`, `abs`, `e^x`; unit grid, panning and zooming |
| Elements | all 118 elements by name, symbol or number, with a periodic-table browser |
| Chat | basic science, everyday knowledge, greetings and a little conversation |
| Honesty | says "I'm not sure" when its own confidence is low (F6 shows the guess) |

It is a very small model, narrow knowledge scope, will frequently produce incorrect answers. Tool results (calculations, conversions, graphs, elements) are computed by code and are reliable if routed correctly.

## Method

### 1. Dataset

- **Tool data** is generated: the *meaning* (an expression tree, a percentage, a conversion, a statistic, a plot function, an element) is sampled first, then phrased through independent choices such as symbols (`9x10`), spoken numbers, request wrappers and keypad typos. Plots are composed from parts so the model learns to copy whole formulas instead of snapping to a few known shapes.
- **Chat data** is a synthetic corpus of 1,900 short facts (physics, chemistry, biology, earth and space, general science, everyday life, conversation), each question was subsequently rewritten by a language model. Answers stay exactly as written, at most 160 characters, with no arithmetic in them. An abstention set teaches "I'm not sure" for trivia.
- **Pretraining text** is about 29M tokens of science and everyday-concept articles ([SimpleWiki](https://huggingface.co/datasets/embedding-data/simple-wiki), SciQ, TinyStories).

The tokenizer is a lowercase byte-level BPE with 1,024 tokens. 

### 2. Training

A pretraining pass (48M tokens) is followed by answer-masked supervised fine-tuning (48M tokens, about 190,000 rows, 40% of the loss on chat) and a short quantisation-aware pass that trains directly in 4-bit weights. Training took under an hour on one RTX 4060Ti. Release builds train on every phrasing and fact; research comparisons use held-out phrasings.

### 3. Native fx-CG50 deployment

The model is a two-layer GRU with 384 hidden units and one causal attention read over the GRU outputs. Weights are 4-bit with per-row scales (about 1.18 MB); activations are 16-bit integers. Decoding uses SH-4A assembly kernels with exact integer LayerNorm and table-based sigmoid, tanh and exp. 

Small repairs run on the model's output before the tool executes (completing list brackets, normalising units, inserting implied multiplication). 

## Model summary

| Property | Value |
| --- | ---: |
| Architecture | GRU 384 x 2 + one causal attention read (64-dim, 2 heads) |
| Parameters | 2,266,368 |
| Vocabulary / context | 1,024 tokens / 260 tokens |
| Weights | 4-bit, ~1.18 MB |
| Generation speed | ~5.8 tok/s stock (measured on hardware) |
| Pretraining / SFT | 48M tokens / 48M tokens |
| Hand-checked maths probe | 95.5% correct |
| Realistic prompt set | 93.4% correct |

The probe sets are isolated from training, but release models trains on all generated phrasings.

## Licences

To be added before release. 
