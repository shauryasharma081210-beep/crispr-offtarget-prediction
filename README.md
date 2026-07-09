**Predicting CRISPR Off-Target Activity from Sequence Features**

**The problem**

CRISPR gene editing works by guiding an enzyme (usually Cas9) to a specific 20-base DNA sequence using a "guide" RNA that matches it. In practice, the guide doesn't just bind its intended target — it can also bind similar-looking sequences elsewhere in the genome, called off-target sites, and edit those by mistake. For CRISPR to be usable in medicine, researchers need to predict which off-target sites are likely to actually get cut, so they can either avoid them or check for them directly.

I worked with a public dataset of ~246,000 CRISPR binding events (on-target and off-target) collected across dozens of experimental techniques (GUIDE-seq, CIRCLE-seq, CHANGE-seq, and others), each row describing a guide sequence, a target site, how many bases mismatch between them, which Cas9 variant was used, and how much editing activity was measured. My goal was to see whether editing activity at off-target sites can be predicted just from properties of the sequence mismatch itself — position, count, and GC content — without needing to run the experiment.

**What I did**

Baseline models. I first trained a random forest to predict editing activity ("Score") directly from mismatch counts and experimental metadata, and a separate classifier to distinguish on-target from off-target sites. Both looked great on paper (R² = 0.91, ROC-AUC ≈ 0.9999) — too great. On closer inspection, both were mostly picking up on trivial shortcuts: which experimental technique was used, and the fact that on-target sites are almost defined as "zero mismatches." These weren't wrong, exactly, but they weren't telling me anything biologically useful, so I flagged them as sanity checks rather than real results and moved on.

Finding and fixing a bug. To get real signal, I wanted to know exactly where along the guide sequence each mismatch occurred — mismatches near the "PAM" (a short motif required for Cas9 to bind) are known to matter more than mismatches farther away. My first attempt compared the guide sequence to the target sequence letter-by-letter, but the two strings turned out to be different lengths (the target sequence includes the PAM, the guide doesn't), so every single comparison failed silently — 0% agreement with the dataset's own reported mismatch counts. I caught this by checking on independent ground truth, switched to a column that marks mismatched bases directly, and got 94% agreement — a working feature set of mismatch position, mismatch count, and GC content.

Building and testing a real model. Using those corrected features, I trained a model on one technique (CHANGE-seq, ~190,000 rows) to predict editing activity. A standard random train/test split gave R² = 0.30 — a modest but plausible result. Before trusting it, I checked how the data was structured and found each of the 111 guides in the dataset appears at up to 61,000 off-target sites. A random split put off-target sites from the same guide in both the training and test sets, so the model could partly just be memorizing guide-specific quirks rather than learning general rules. I re-ran the evaluation using a split that keeps every guide entirely in either training or test, never both — R² dropped to about 0.02.

**What I found**

The honest result is that mismatch position, mismatch count, and GC content alone are not enough to predict CRISPR off-target activity for a guide the model hasn't seen before. The encouraging-looking 0.30 was mostly an artifact of data leakage, not real predictive power. That's a useful (if less flattering) finding: it suggests off-target activity depends on something more specific to each guide or target site — like local chromatin structure, sequence context beyond the 20 bases, or epigenetic factors — that these four features don't capture.

**What I learned**

The biggest lesson wasn't about CRISPR biology, it was about not trusting a good-looking number. Three separate results in this project looked strong at first (R² = 0.91, AUC ≈ 0.9999, R² = 0.30) and all three turned out to be inflated by some form of information leaking from the training data into the evaluation. Catching that required checking assumptions I hadn't stated out loud — that a length check would actually run, that a random split was really "random" with respect to what I cared about. That habit of asking "why does this look this good?" is the part of this project I'd want to carry into any future research.

**Possible next steps**


Add sequence-context features beyond the 20-base guide (flanking DNA, local GC skew)
Incorporate chromatin accessibility or epigenetic data, which the dataset filename (allframe_update_addEpige) suggests may be available
Try the grouped-split evaluation across other techniques (GUIDE-seq, CIRCLE-seq) to see if the same leakage pattern holds
Compare against published off-target scoring tools (CFD score, Elevation, CRISPOR) as a benchmark
