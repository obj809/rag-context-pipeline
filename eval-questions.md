# EPBC Act eval questions

The retrieval-eval ground truth for the RAG pipeline, over the three-volume
**Environment Protection and Biodiversity Conservation Act 1999**. The machine-readable
source is `engine-rag-context-pipeline/eval/dataset.jsonl`; this file is the
human-readable version.

Each question lists its expected answer and the `(volume, page)` location(s) in the Act
where the answer text sits — the relevance label `eval/run_eval.py` scores against.
Pages are PDF page numbers (which differ from the Act's printed page numbers) and
restart per volume, so the citation is always volume + page.

Baseline on the current index: **hit-rate@1 50%, hit-rate@3/6/10 ≈ 92%, MRR ≈ 0.65.**

| # | Question | Answer | Source |
|---|----------|--------|--------|
| 1 | What are the objects of the Environment Protection and Biodiversity Conservation Act 1999? | Protect the environment (especially matters of national environmental significance), promote ecologically sustainable development, conserve biodiversity, protect and conserve heritage, promote a co-operative approach with the States and others, and recognise the role of Indigenous people. | s 3 — Vol 1, p.24–25 |
| 2 | What are the matters of national environmental significance protected under Part 3? | Nine matters: declared World Heritage properties, National Heritage places, declared Ramsar wetlands, listed threatened species and ecological communities, listed migratory species, nuclear actions, Commonwealth marine areas, the Great Barrier Reef Marine Park, and water resources in relation to coal seam gas and large coal mining development. | Part 3 — Vol 1, p.35, 110–112 |
| 3 | What does the Act require before taking an action that may significantly affect a declared World Heritage property? | Approval is required: a person must not take an action that has, will have, or is likely to have a significant impact on the world heritage values of a declared World Heritage property without an approval under Part 9. | s 12 — Vol 1, p.35–36 |
| 4 | What is the civil penalty for taking an action with a significant impact on a listed threatened species without approval? | 5,000 penalty units for an individual and 50,000 penalty units for a body corporate. | s 18 — Vol 1, p.54–55 |
| 5 | What is a nuclear action under the Act? | Actions such as establishing or operating a nuclear installation (e.g. a reactor, enrichment or reprocessing plant), mining or milling uranium ore, and transporting or managing radioactive waste. | s 22 — Vol 1, p.63–64 |
| 6 | When does a coal seam gas or large coal mining development need approval under the Act? | When the development has, will have, or is likely to have a significant impact on a water resource (the "water trigger"). | s 24D — Vol 1, p.78–79 |
| 7 | What protection applies to the Great Barrier Reef Marine Park? | Approval is required before an action that has, will have, or is likely to have a significant impact on the Marine Park; the civil penalty is 5,000 penalty units for an individual and 50,000 for a body corporate. | s 24B — Vol 1, p.74–75 |
| 8 | How does the Act protect a declared Ramsar wetland? | A person must not take an action that has, will have, or is likely to have a significant impact on the ecological character of a declared Ramsar wetland without approval under Part 9. | s 16 — Vol 1, p.49–50 |
| 9 | What is the object of a bilateral agreement under Part 5? | To protect the environment, promote ecologically sustainable development, and ensure an efficient, timely and effective process for environmental assessment and approval while minimising duplication between the Commonwealth and a State or Territory. | s 44 — Vol 1, p.137 |
| 10 | Who must refer a proposal to take an action that may be a controlled action, and to whom? | A person proposing to take an action that they think is or may be a controlled action must refer the proposal to the Minister. | s 68 — Vol 1, p.167–168 |
| 11 | What is the Australian Whale Sanctuary? | An area established to give formal protection to whales and other cetaceans, comprising the Commonwealth marine areas and prescribed waters; killing, injuring, taking or interfering with a cetacean in the Sanctuary is an offence. | ss 225–226 — Vol 1, p.438–439 |
| 12 | What is a conservation agreement under the Act? | A voluntary agreement the Minister may enter into, on behalf of the Commonwealth, with a person for the protection and conservation of biodiversity (and other matters protected by Part 3). | s 305 — Vol 2, p.159–160 |

## Known retrieval miss

Question 1 (**objects of the Act**, s 3) is a consistent miss in the top-k: the phrase
"objects of this Part" recurs throughout the Act, so the s 3 page doesn't surface above
the other "objects" provisions. It's kept in the set as an honest weakness — a good
canary for whether hybrid/keyword search would help.
