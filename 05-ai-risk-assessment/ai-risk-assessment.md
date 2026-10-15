# AI Tool Justification and Risk Assessment

### PassGAN: A Generative Adversarial Network for Password Guessing

**Bincom Training Program Assignment**

**Prepared by**: CHUKWU PRAISEGOD

**Date**: September 21, 2026

---

## 1. Tool Selection and Justification

This assessment examines PassGAN, a password-guessing tool that uses a Generative Adversarial Network (GAN) to learn the statistical distribution of real, human-chosen passwords directly from leaked password datasets, then generates new password candidates that follow that same distribution. PassGAN was selected for this assessment because it connects directly to the brute-force exercise carried out earlier in this assignment: the SSH attack in this project used THC-Hydra with a wordlist manually derived from the `rockyou.txt` leaked-password corpus, whereas PassGAN represents the AI-driven evolution of exactly that same underlying idea: learning from leaked password data rather than relying on manually curated wordlists or hand-written mangling rules. Assessing PassGAN therefore allows a direct, informed comparison between traditional dictionary-based brute forcing (as performed in this project) and an AI-augmented equivalent.

## 2. Tool Overview

### 2.1 Origin and Purpose

PassGAN was introduced by Hitaj et al. in the 2017 paper *"PassGAN: A Deep Learning Approach for Password Guessing,"* as the first password-guessing technique built on generative adversarial networks. Its stated goal was to replace the labor-intensive, expert-driven process of writing password mangling rules (used by tools such as HashCat and John the Ripper) with a model that learns password patterns automatically, directly from real leaked password data, without any manual analysis or built-in assumptions about how people construct passwords.

### 2.2 How It Works

PassGAN is built on an Improved Wasserstein GAN architecture, consisting of two competing neural networks trained together: a **generator**, which learns to produce password-like strings from random noise, and a **discriminator**, which learns to distinguish generated strings from real passwords in the training data. As training progresses, the generator's output converges toward the statistical distribution of the real password corpus it was trained on, meaning it increasingly produces plausible, human-like passwords that were never explicitly programmed as rules, purely by having learned the underlying patterns in real leaked data.

### 2.3 Demonstrated Effectiveness

In its original evaluation, PassGAN outperformed John the Ripper's rule-based generation by roughly a factor of two and performed competitively with HashCat's rules (within a factor of two), despite using no prior domain knowledge of password structure. More significantly, when PassGAN's output was combined with HashCat's own rule-based guesses, the combined approach matched substantially more real passwords than HashCat alone, with independent replications reporting gains in the range of roughly 18% to 73% more matches depending on dataset and configuration. This indicates PassGAN's value is less about replacing existing tools and more about covering a portion of the password space that manually written rules miss entirely.

## 3. Legitimate Uses and Capabilities

- Red team and penetration testing engagements, to realistically model how quickly an organization's actual user-chosen passwords could be guessed by a well-resourced attacker.
- Corporate and institutional password policy auditing: testing whether current complexity requirements meaningfully resist AI-driven guessing, not just traditional dictionary attacks.
- Academic and industry security research into password selection behavior, password entropy estimation, and the ongoing arms race between password policies and guessing techniques.
- Defensive password-strength meters and breach-screening tools, which can incorporate GAN-style modeling to warn users when a chosen password is close to a commonly-guessable pattern, even if the exact password has not previously appeared in a known leak.

## 4. Misuse Potential

PassGAN's core innovation, removing the need for manual rule engineering, is precisely what also lowers the barrier to effective misuse. A threat actor no longer needs specialized expertise in password pattern analysis to generate high-quality guesses; they need only a trained model and a sufficiently relevant leaked-password dataset, both of which are realistically obtainable given the volume of historical credential breaches publicly circulating.

- **Credential stuffing and account takeover:** AI-generated candidate passwords can be fed into the same automated login tools already used for traditional brute-force attacks (as demonstrated with Hydra in this project), meaning the practical attack surface and delivery mechanism are unchanged, with only the quality of the guesses improving.
- **Targeted attacks against specific organizations or individuals:** if trained or fine-tuned on a dataset relevant to a specific population (e.g. a past breach of a related service), PassGAN-style models could generate guesses tailored to that population's password habits.
- **Erosion of traditional password policy assumptions:** because PassGAN learns from genuine human password choices rather than following hand-written complexity rules, passwords that satisfy typical policy requirements (length, character classes) may still fall within patterns the model has learned to predict.
- **Low attribution and low cost:** like traditional wordlist attacks, AI-generated guessing produces no inherently distinguishing network signature, making it hard to differentiate from conventional brute-force traffic at detection time.

## 5. Risk Assessment

| Risk | Likelihood | Impact | Notes |
|---|---|---|---|
| Credential stuffing / account takeover using AI-generated guesses | High | High | PassGAN-generated candidates can be fed directly into existing cracking pipelines (e.g. Hydra, HashCat), so this risk compounds existing brute-force exposure rather than requiring new attacker tooling. |
| Lowered technical barrier to effective password guessing | High | Medium | Removes the need for manual rule engineering; a less skilled attacker can achieve results previously requiring specialist knowledge. |
| Reduced effectiveness of traditional complexity-based password policies | Medium | High | PassGAN learns real human password patterns directly, so policies designed around human-guessable rules may not resist a model trained on similar leaked data. |
| Amplification when combined with existing cracking tools | Medium | High | Independent research combining PassGAN's output with HashCat's rules matched substantially more passwords than HashCat alone, indicating a compounding rather than replacement effect. |
| Misuse of leaked-password training data / privacy exposure | Medium | Medium | Training relies on real breached credential corpora; redistribution or careless handling of such training data carries its own privacy and legal exposure. |
| False sense of security from network-level detection alone | Low | Medium | AI-generated guesses are submitted over the same protocols (e.g. SSH) as traditional dictionary attacks, so SIEM detection built for brute-force patterns (as demonstrated in this project) still applies, but should not be assumed sufficient on its own. |

## 6. Mitigations and Recommendations

- Enforce multi-factor authentication (MFA) everywhere feasible: this remains effective regardless of whether the underlying password guess originated from a traditional wordlist or an AI-generated one, since it removes password-only compromise as a sufficient condition for account takeover.
- Move toward passwordless or hardware-backed authentication (e.g. FIDO2/WebAuthn) for high-value accounts, eliminating the password-guessing attack surface entirely rather than trying to outpace increasingly capable guessing techniques.
- Screen new passwords against both traditional breach-derived blocklists and, where feasible, generative-model-aware pattern detection, rather than relying solely on static complexity rules that AI-driven guessing has shown can still be satisfied by predictable choices.
- Maintain rate limiting, account lockout, and connection throttling on authentication services, as this project's own SSH daemon throttling incidentally demonstrated, these controls disrupt automated guessing regardless of whether the guesses are AI-generated or dictionary-based.
- Continue SIEM-based detection of authentication failure patterns (as built in this project's Wazuh deployment), since AI-generated password guesses are submitted through the same authentication protocols as traditional attacks, existing brute-force correlation rules (such as the Rule 40111/40112 pattern observed in the incident analysis) remain a valid detection layer and should not be assumed obsolete against AI-augmented attacks.
- Restrict and audit access to internal credential/leak datasets, and apply data-handling controls to any password-research activity, given that tools in this category depend on and interact with real breached personal data.

## 7. Conclusion

PassGAN doesn't defeat existing defenses outright: it automates a skill that once required manual expertise, while still attacking through the same channel (automated login attempts) that current detection already covers. So the right response isn't a control built specifically for AI-generated guessing, but stronger fundamentals: MFA, rate limiting, SIEM-based anomaly detection, and a longer-term move away from passwords alone.
