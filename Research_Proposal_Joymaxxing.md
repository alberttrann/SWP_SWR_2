| Research title (English) | LLM-Assisted Requirements Elicitation for AI/ML Outsourcing Projects: An AI-Project-Specific Taxonomy and Requirement Coverage Framework |
| ------------------------ | ---------------------------------------------------------------------------------------------------------------------------------------- |
| Sub-committee            | IT                                                                                                                                       |
| Group name               | JoyMaxxing                                                                                                                                        |
| Authors                  | Trần Võ Minh Hùng; Huỳnh Tuấn Khang; Chiêm Minh Thức; Võ Cao Minh; Bùi Phạm Chí Nhân                                                     |
| Mentor                   | Thân Thị Ngọc Vân                                                                                                                        |


## ABSTRACT

The rapid adoption of artificial intelligence (AI) across industries has created a growing market for outsourced AI development services. However, a critical structural gap persists: non-technical decision-makers who commission AI projects cannot formulate requirements that are complete enough for AI/ML experts to scope and implement work correctly. This mismatch - rooted in the absence of a domain-specific elicitation methodology - produces systematic project failures, scope creep, and mismatched expert-client pairing in AI service marketplaces.

This research investigates how large language models (LLMs) can support requirements elicitation specifically for AI/ML projects, addressing a gap unresolved by both traditional requirements engineering (RE) theory and the emerging LLM4RE literature. The study develops three interconnected contributions: (1) an AI-project-specific requirement dimension taxonomy covering data availability, evaluation metrics, non-determinism, cost-accuracy trade-offs, LLMOps concerns, and expertise intersection requirements; (2) an LLM-based elicitation assistant that transforms vague, symptom-level problem descriptions into structured AI-SDLC-aligned specifications through multi-round interactive dialogue; and (3) a Requirement Coverage Score (RCS) metric for measuring elicitation completeness against expert-annotated ground truth.

The research employs a mixed-methods experimental design comparing LLM-assisted elicitation against unaided, checklist-based, and human-interviewer conditions across 15–20 real-world AI project cases. Findings will establish whether LLM-assisted elicitation can significantly improve requirement coverage, identify missing stakeholder perspectives, and correctly classify AI project types - capabilities essential for effective AI marketplace platform design.

**Keywords:** requirements elicitation, large language models, AI/ML software engineering, AI marketplace, requirement coverage, LLM4RE, non-technical stakeholders, software development lifecycle

---

## 1. INTRODUCTION

### 1.1 Literature Review

#### Overview and Thematic Structure

The literature relevant to this research organizes into four thematic clusters: (i) traditional requirements elicitation theory, (ii) NLP and LLM applications in requirements engineering, (iii) LLMs in software engineering tasks broadly, and (iv) software engineering for AI/ML projects. Together, these bodies of work converge on a gap: no existing elicitation methodology is designed for AI-project requirements, particularly in the context of outsourced or marketplace-based AI service commissioning where non-technical buyers interact with domain experts.

---

#### Cluster 1: Traditional Requirements Elicitation - Foundations and Persistent Challenges

Requirements elicitation has been recognized as the most critical and failure-prone activity in the software development lifecycle for over three decades. Zowghi and Coulin (2005) present the canonical survey of elicitation techniques, covering interviews, questionnaires, ethnography, protocol analysis, goal-based approaches, and scenario-based methods. Their central finding is that no single technique suffices for all contexts, and effective elicitation invariably requires a combination of complementary approaches calibrated to the stakeholder population and problem domain. Critically, they identify that elicitation fails not because of inadequate techniques in isolation, but because of what they term the *communication gap* - the fundamental semantic difference between the problem-owning and problem-solving communities. Stakeholders often cannot articulate their own needs, conflate solutions with requirements, overlook tacit knowledge, or resist change. These barriers remain unsolved by any purely technical elicitation approach.

Nuseibeh and Easterbrook (2000) frame requirements engineering as a multi-disciplinary activity drawing on cognitive science, sociology, linguistics, and formal methods. They establish that stakeholders have goals that vary, conflict, and may not be explicit or even articulable - and that the requirements engineer must navigate this complexity across multiple iterative sessions rather than extracting requirements in a single pass. Their roadmap explicitly anticipates one of the central challenges this research addresses: the need for practitioners who possess both social skills to interact with non-technical customers *and* technical skills to communicate with system designers. This dual competency requirement is rarely available in a single person, and its absence systematically degrades elicitation quality in AI outsourcing contexts.

Goguen and Linde (1993) extend the theoretical foundation with an ethnomethodological perspective, distinguishing between what stakeholders *say* they need, what they *show* through behavior, and what they *actually* need operationally. This three-way distinction is directly applicable to non-technical AI service buyers, who consistently describe symptoms (e.g., "the AI output is inconsistent and costs too much") rather than problems, and propose solutions (e.g., "we want to fine-tune a model") rather than requirements - without the technical context needed to evaluate whether those solutions are appropriate.

**Identified Gap from Cluster 1:** All foundational RE theory assumes a relatively stable, articulable problem domain. The client is understood to know they are building accounting software or a hospital system - the domain is known, even if requirements are incomplete. None of these frameworks account for the structurally novel case where the client does not know what type of technical solution is needed, cannot distinguish between AI approaches (automation vs. fine-tuning vs. pipeline architecture vs. RAG), and is not even engaging with the right internal stakeholder. This is the challenge this research addresses.

---

#### Cluster 2: NLP and LLM Applications in Requirements Engineering

The application of natural language processing to requirements engineering has a history extending back to the early 1990s. Ferrari et al. (2017) provide the foundational survey of NLP-in-RE, establishing that NLP has been applied to requirements classification, ambiguity detection, completeness checking, conflict identification, and traceability - primarily operating on existing requirements documents rather than on the upstream conversations that produce them. Their conclusion that "the best is yet to come" correctly anticipated the transformative impact of LLMs.

The most current and comprehensive systematic literature review of LLMs specifically for requirements engineering is Zadenoori et al. (2025), analyzing 74 primary studies published between 2023 and 2024. Their findings document a paradigm shift from earlier NLP4RE work: whereas traditional NLP was applied primarily to defect detection and classification, LLM4RE research concentrates on cognitively demanding tasks - requirements elicitation (20% of studies) and requirements validation (20% of studies) - alongside requirements modeling (12%) and broader software engineering tasks (15%). The authors note that this shift is not merely a performance improvement but a qualitative expansion in what is now computationally tractable. Importantly, they find that interactive prompting strategies - the most natural modality for an elicitation conversation - appear in only 4% of studies, and Retrieval-Augmented Generation (RAG)-based approaches in only 7%, representing the highest-potential underexplored territory for elicitation assistant design. Research still skews heavily toward laboratory experiments (76% of studies) with limited real-world field validation, and prompt engineering practices are largely underdocumented.

Hemmat et al. (2025), in a systematic mapping study of 29 primary studies published between 2020 and 2024, similarly find that requirements elicitation and analysis integrate LLMs most effectively. Their analysis identifies GPT-family models as most commonly employed, with contextual prompting (14 studies) and few-shot prompting (11 studies) as particularly effective strategies. They document domain-specific knowledge challenges as the primary limitation - LLMs hallucinate and produce inaccurate output precisely because they are trained as generalists, and the absence of domain-specific context degrades performance on specialized requirements tasks. Their evaluation framework identifies correctness-and-completeness metrics and precision-and-recall as the dominant assessment approaches, but notes the absence of standardized benchmarks for elicitation quality - a gap this research addresses through the Requirement Coverage Score metric.

Arora, Grundy, and Abdelrazek (2023) provide both a SWOT analysis of LLMs across all RE stages and a preliminary empirical evaluation on a real-world application (ActApp, a diabetes management system for Type 2 diabetes patients). Their empirical results are particularly instructive: ChatGPT-assisted elicitation by an experienced requirements engineer achieved 82% precision and 58% recall against 20 ground-truth requirements, substantially outperforming novice users (15–29% precision, 20% recall). This finding establishes the critical role of domain expertise in guiding LLM elicitation: the LLM amplifies the domain-knowledgeable user rather than replacing them. Their SWOT analysis identifies IP loss, data security, and privacy as key threats - identifying that eliciting requirements via public LLM APIs may expose sensitive system architecture details before any contract is signed, a concern that directly surfaces in the AI marketplace context this research addresses.

Luitel, Hassani, and Sabetzadeh (2024) demonstrate that LLMs can improve requirements completeness by automatically identifying missing information in natural language requirements, achieving meaningful precision and recall improvements over baseline methods. This completeness-identification capability is a primary function the elicitation assistant in this research must perform: given a vague problem description, the assistant must identify what information is absent before it can be provided.

**Identified Gap from Cluster 2:** All 74 studies surveyed by Zadenoori et al. (2025) operate on requirements documents - structured or semi-structured text that already exists. None studies the upstream elicitation conversation that produces those documents, particularly from vague, symptom-level problem descriptions from non-technical clients. The most closely related empirical work (Arora et al., 2023) operates on an already-understood medical application domain. No study addresses AI-project-specific requirements elicitation from non-technical founders in an outsourced service context.

---

#### Cluster 3: LLMs in Software Engineering - Capabilities and Design Patterns

Fan et al. (2023) survey LLMs across software engineering tasks including coding, design, and requirements, establishing that LLMs demonstrate strong capability in code generation, test case generation, and requirements specification, while hallucination, domain knowledge gaps, and context window limitations remain persistent challenges. This broad capability baseline establishes the technical plausibility of LLM-based elicitation.

White et al. (2023, 2024) develop a prompt pattern catalog specifically for software engineering tasks, proposing structured patterns directly applicable to elicitation assistant design. The Cognitive Verifier Pattern (the LLM asks clarifying sub-questions before producing a final answer) models the iterative probing behavior an elicitation assistant needs. The Question Refinement Pattern (the LLM suggests better phrasings of the user's question) enables non-technical users to articulate needs more precisely. The Specification Disambiguation Pattern (the LLM identifies ambiguous areas and proposes clarifications) directly maps to ambiguity detection in vague AI project descriptions. These patterns provide a principled basis for the elicitation assistant's prompt engineering design.

Vogelsang and Fischbach (2024) provide systematic guidelines for using LLMs in natural language processing tasks for requirements engineering, emphasizing that few-shot prompting with domain-specific examples substantially improves output quality over zero-shot approaches. Their guidelines inform the few-shot design of the elicitation assistant, which will use documented AI project cases as exemplars.

Cheng et al. (2024) conduct a systematic literature review of generative AI in RE processes, highlighting practical challenges in implementation that point toward the need for domain-specific tooling rather than generic LLM applications. Their identification of hallucination and implementation inconsistency as key limitations motivates the hybrid human-in-the-loop design of the research's elicitation assistant.

**Identified Gap from Cluster 3:** The LLM-in-SE literature has focused on code generation, test generation, and formal specification. Requirements elicitation - particularly the upstream, conversational, domain-agnostic elicitation of vague problem descriptions - is underrepresented. No domain-specific elicitation taxonomy for AI projects exists in the literature, and no study proposes domain-specific evaluation metrics for AI project elicitation quality.

---

#### Cluster 4: Software Engineering for AI/ML Projects - The Distinct SDLC

The foundational empirical work establishing that AI/ML projects have a fundamentally different development lifecycle from traditional software is Amershi et al. (2019), a Microsoft Research case study of 551 software engineers across the company. Their study identifies three fundamental differences: (1) data discovery, management, and versioning is substantially more complex than equivalent activities in traditional software engineering; (2) model customization and reuse require deep ML expertise well beyond standard software engineering skills; and (3) ML components are more difficult to maintain as distinct modules due to component entanglement and non-monotonic error propagation. Across all experience levels surveyed, data availability, collection, cleaning, and management is the top-ranked challenge - directly relevant to the data engineering and data analysis prerequisites that non-technical founders routinely omit from their AI project descriptions. Amershi et al. also find that requirements specification becomes a more prominent challenge as AI experience increases: experienced practitioners allocate significantly more effort to formal specification than novices, suggesting that the requirements problem is not eliminated by technical expertise but rather becomes more sharply defined as the practitioner understands what information the system actually needs.

Sculley et al. (2015), in their landmark analysis of hidden technical debt in machine learning systems, identify AI-specific debt forms not recognized by traditional software engineering frameworks: undeclared consumers (components that secretly depend on ML outputs), data dependency rot (stale feature pipelines), feedback loops, and configuration complexity. For requirements elicitation purposes, these concerns must be identified and specified during the requirements phase, not retrofitted during system maintenance - yet no existing requirements elicitation framework includes these as first-class dimensions. The absence of observability requirements, structured output schema requirements, and evaluation metric definitions from non-technical buyer descriptions consistently leads to the kinds of operational failures that Sculley et al. document at production scale.

Ahmad et al. (2023) establish a requirements engineering framework for human-centered AI systems that explicitly includes bias, ethical considerations, and the integration of non-deterministic AI components in larger software systems as requirements-level concerns. Their framework extends traditional RE into the AI domain but does not address the elicitation context specifically - how to surface these concerns from a non-technical client who does not know they exist.

**Identified Gap from Cluster 4:** While the AI/ML SE literature clearly establishes that AI projects have unique development characteristics incompatible with traditional requirements frameworks, no elicitation methodology has been designed to capture these characteristics from non-technical clients. The gap is not merely theoretical: the case evidence motivating this research demonstrates concretely that non-technical AI service buyers systematically omit AI-specific requirement dimensions - evaluation metrics, data availability, non-determinism handling, LLMOps concerns, and expertise intersection requirements - from their initial problem descriptions.

---

#### Summary of Key Findings and Implications

The literature review reveals a four-way convergence of gaps at the intersection of requirements engineering, LLMs, and AI project development:

**Finding 1:** Requirements elicitation failure in AI projects is a domain-alignment problem, not a technology problem. Existing LLM applications to elicitation assume the domain is understood and requirements are partially specified. The novel challenge of transforming symptom-level descriptions from someone who cannot articulate the technical domain has not been studied. This gap motivates the upstream focus of this research.

**Finding 2:** AI/ML projects have unique requirement dimensions absent from any existing elicitation framework. Data availability, evaluation metric definition, non-determinism handling, cost-accuracy-speed trade-offs, LLMOps concerns, and expertise intersection requirements are routinely omitted by non-technical buyers and not captured by any existing elicitation tool or template. This gap motivates the AI-project-specific taxonomy contribution.

**Finding 3:** Interactive and RAG-based prompting strategies - the most natural modalities for elicitation dialogue - appear in only 4% and 7% of LLM4RE studies respectively, representing the highest-potential and least-explored opportunity for elicitation assistant design. This gap motivates the assistant's multi-round interactive dialogue mechanism.

**Finding 4:** The LLM4RE field lacks domain-specific elicitation benchmarks and standardized evaluation metrics. The Requirement Coverage Score (RCS) metric proposed in this research fills this gap and provides a measurable criterion for elicitation quality that can be generalized to other AI project elicitation contexts. This gap motivates the evaluation design.

**Implications for future research and practice:** This research establishes a methodological foundation for AI-specific requirements elicitation that future work can extend to other specialized domains. Practically, the elicitation taxonomy and assistant design can be embedded directly into AI marketplace platforms to improve client-expert matching accuracy, reduce pre-contract miscommunication, and lower the risk of scope creep and project failure. The dual-persona problem identified - the systematic divergence between what non-technical decision-makers describe and what technical implementers know - is a structural feature of AI outsourcing that any AI marketplace platform must design for.

---

### 1.2 The Necessity of the Research

#### Problem Definition

The global AI services market is expanding at an accelerating pace, yet the mechanisms through which AI projects are commissioned, scoped, and matched to expertise remain unchanged from those designed for traditional software projects. This structural mismatch produces a predictable, empirically documented failure pattern: non-technical decision-makers describe AI problems in symptom-level, domain-agnostic language; this description is insufficient to scope the actual technical work required; mismatched experts are engaged based on inadequate information; and projects subsequently fail, expand beyond their original contracts, or require costly post-hoc scope revisions. The problem is not that clients are unintelligent or experts are negligent - it is that no methodological bridge exists to translate between the client's experiential description of business pain and the expert's need for technically grounded requirement specifications.

The specific, observable, and currently unresolved problem this research addresses is: *non-technical AI service buyers cannot formulate requirements that are complete enough for AI/ML experts to scope and implement AI projects correctly, and no existing elicitation tool or method has been designed to close this gap by transforming vague, symptom-level problem descriptions into AI-SDLC-aligned technical specifications.*

This research tests the hypothesis that an LLM-based elicitation assistant, guided by an AI-project-specific requirement dimension taxonomy, can systematically surface the AI-project requirements that non-technical buyers fail to articulate - doing so more completely and in less time than unassisted or traditionally-assisted elicitation approaches.

#### Originality

This research makes three original contributions not present, individually or in combination, in any existing published work.

**Original Contribution 1: An AI-Project-Specific Requirements Elicitation Taxonomy.** This research defines and validates a requirement dimension taxonomy specific to AI/ML projects - one that includes dimensions absent from all existing RE frameworks: data availability and quality assessment, evaluation metric and ground truth definition, AI problem type classification (distinguishing automation flows, fine-tuning tasks, RAG system design, agent systems, and full pipeline architecture), cost-accuracy-speed constraint specification, non-determinism handling requirements, structured input/output and observability design requirements, expertise intersection identification, and LLMOps operational concerns. This taxonomy is an original contribution independent of its application in the LLM elicitation assistant.

**Original Contribution 2: The Upstream Elicitation Conversation as the Unit of Analysis.** All 74 studies identified in Zadenoori et al.'s (2025) systematic review operate on requirements documents - artifacts that already exist. This research focuses on the elicitation conversation that precedes those documents: the transformation of a vague, symptom-level problem description into a structured requirement set. This is a harder and less-studied problem because the input is unstructured, the domain is implicit, and the client often does not know what information is needed. The research is positioned at the most upstream and highest-leverage point in the AI project lifecycle.

**Original Contribution 3: The Dual-Persona Elicitation Problem.** This research identifies and operationalizes a structural feature of AI project elicitation that has not been studied in any existing work: the systematic divergence between what non-technical decision-makers can describe and what technical implementers know about their own systems. In the motivating case documented in the research materials, the CEO's description ("AI output is inconsistent, cost $60/day, want to fine-tune a model") diverged completely from what the tech team eventually disclosed (a distributed Kafka-Go-Python pipeline with Milvus vector search, 600,000 legacy records, and semantic landing page deduplication as a newly required add-on phase). Any elicitation methodology treating "the client" as a single coherent information source will systematically fail for AI projects. This research designs and evaluates an elicitation assistant that explicitly accounts for this dual-persona structure.

#### Relevance and Scientific Significance

*Industrial relevance* is directly evidenced by the documented case motivating this research. The communication failure between a non-technical AdTech CEO and an AI/ML consultant - requiring multiple meetings, a three-week ghosting incident following premature disclosure of detailed technical proposals, a contract revision due to late-stage architecture changes, and a separately negotiated add-on phase before actual technical requirements were fully known - represents a pattern occurring routinely across the AI services industry. Platforms such as Upwork and Fiverr report AI-related projects among their fastest-growing categories, yet no specialized tooling exists for AI project scoping or expert-client matching.

*Academic relevance* follows from the four-way convergence of gaps established in the literature review. The combined absence of: a domain-specific AI elicitation taxonomy, a study of upstream elicitation conversations for AI projects, research on the dual-persona problem in AI commissioning, and field-validated elicitation benchmarks creates a clearly defined research space where new knowledge is needed and where this study's contributions are additive rather than duplicative. The 136% year-on-year growth in LLM4RE publications identified by Zadenoori et al. (2025) confirms this as an area of rapidly accelerating community interest.

*Vietnamese market relevance* is specific and underserved. Vietnam's technology sector is characterized by a large and growing SME-level AI adopter base, a concentration of AI technical talent in freelance and consulting arrangements rather than large enterprise structures, and a predominantly non-technical founder class driving AI adoption decisions. This combination means the elicitation gap is acutely felt in the Vietnamese context - as directly evidenced by the case involving a Vietnamese AdTech company and a Vietnamese AI consultant - yet Vietnamese-language AI requirements elicitation has received essentially no academic attention. This research contributes domain-specific knowledge relevant to the Vietnamese AI services context while producing findings generalizable to comparable emerging markets.

#### Research Questions and Hypotheses

**RQ1:** How effectively can an LLM-based requirement elicitation assistant surface AI-project-specific requirements from vague, non-technical problem descriptions, compared to traditional interview-based and unaided approaches?

**RQ1.1:** Can the assistant generate AI-domain-specific clarifying questions that cover the dimensions a complete AI project specification requires?

**RQ1.2:** Can the assistant identify missing stakeholder perspectives - specifically the gap between what a non-technical decision-maker describes and what the technical team knows?

**RQ1.3:** Does LLM-assisted elicitation reduce the time to achieve complete requirement coverage compared to human interviewer and unaided baselines?

**RQ1.4:** Can the assistant correctly classify the type of AI work implied by a vague description - distinguishing among fine-tuning tasks, API integration, data engineering prerequisites, pipeline architecture, and automation flows?

**Hypothesis H1 (Coverage):** An LLM elicitation assistant guided by an AI-project-specific requirement dimension taxonomy will achieve significantly higher Requirement Coverage Score than unaided developer elicitation and rule-based checklist approaches when applied to vague, non-technical AI project descriptions.

**Hypothesis H2 (Dual-Persona):** The LLM elicitation assistant will identify missing stakeholder perspectives - specifically the divergence between non-technical decision-maker descriptions and technical team knowledge - at a rate significantly higher than human interviewer approaches operating without domain-specific guidance.

**Hypothesis H3 (Efficiency):** LLM-assisted elicitation will achieve equivalent or higher Requirement Coverage Score in significantly less elapsed time than structured human interview approaches.

**Hypothesis H4 (Classification):** The elicitation assistant will correctly classify the type of AI work implied by a vague description at a rate significantly above chance, enabling more accurate expert-client matching.

---

### 1.3 Feasibility of Research

#### Research Question

The research question is clearly bounded and testable: whether an LLM-based elicitation assistant guided by an AI-project-specific taxonomy can improve requirement coverage for AI projects compared to existing approaches. The question decomposes into four measurable sub-questions and four falsifiable hypotheses (H1–H4 above), each mapping to a specific experimental condition in the study design. The research question does not require access to proprietary datasets, expensive specialized equipment, or a large participant pool - it requires documented AI project cases, domain expert annotators, and a small sample of non-technical participants for think-aloud sessions, all of which are accessible within the project timeline.

#### Research Design

The study employs a mixed-methods experimental design with five phases:

**Phase 0 - Taxonomy Development:** Literature review, expert interviews (n=5–8 AI consultants), and case analysis of the motivating AI project case to derive the AI-project-specific requirement dimension taxonomy. Output: a validated taxonomy with dimensions, sub-dimensions, and scoring criteria.

**Phase 1 - Dataset Construction:** Collection of 15–20 AI project cases, each with (a) an initial vague problem description (analogous to a Facebook post or brief client inquiry) and (b) expert-annotated ground truth requirements covering all taxonomy dimensions. The motivating case (AdTech company ad verification pipeline) provides Case 1 with unusually rich ground truth documentation. Additional cases will be sourced from publicly available AI project postings and synthetic cases constructed from real project types.

**Phase 2 - Baseline Conditions:** Three baseline conditions against which the assistant is compared: (i) unaided developer elicitation - developer reads the description and writes requirements directly; (ii) human structured interview - a requirements engineer conducts a semi-structured interview using a standard template; (iii) rule-based checklist - a domain-agnostic completeness checklist applied to the description.

**Phase 3 - Elicitation Assistant Implementation:** Development of the LLM-based elicitation assistant incorporating the taxonomy dimensions, multi-round interactive dialogue, and few-shot prompting with documented AI project cases as exemplars. The assistant will be implemented using the Claude API with structured prompting following White et al.'s (2023) cognitive verifier and question refinement patterns.

**Phase 4 - Quantitative Evaluation:** Comparison of all four conditions on Requirement Coverage Score, elicitation time, question relevance precision, client comprehensibility rating, and AI taxonomy classification accuracy.

**Phase 5 - Qualitative Layer:** Think-aloud sessions with non-technical participants interacting with the assistant; thematic analysis of expert interviews; case study reconstruction of information emergence timeline from the motivating case.

#### Sample Size and Population

**Case dataset:** 15–20 AI project cases with expert-annotated ground truth. Cases will be selected to represent diversity across AI problem types (ML pipeline, LLM application, automation flow, data engineering, fine-tuning), industries (AdTech, HealthTech, EdTech, FinTech), and scale (startup, SME, enterprise). Inclusion criteria: case must have a documented initial description from a non-technical buyer and sufficient technical documentation to establish ground truth. Exclusion criteria: cases where the client is technically sophisticated (engineering background) or where the initial description is already structured.

**Human participants for baseline conditions:** 5–8 participants per baseline condition (15–24 total), drawn from software engineering students (junior developers for the unaided condition) and junior project managers (for the structured interview condition). Inclusion criteria: less than 2 years of AI/ML project experience. Exclusion criteria: prior experience specifically in requirements engineering for AI systems.

**Expert annotators for ground truth:** 3 domain experts per case (AI consultants or senior ML engineers with 3+ years of AI project experience), used to establish inter-rater reliability (target Cohen's kappa ≥ 0.70) for ground truth annotation.

**Think-aloud participants:** 5–8 non-technical founders or product managers with no engineering background, recruited through university entrepreneurship programs and startup communities.

#### Data Sources

The study draws on four data source types:

1. **Existing case documentation:** The motivating case (AdTech ad verification pipeline) provides a fully documented primary case including the initial Facebook post, messenger conversations, pre-project proposal, contract, WBS documents, and add-on phase specifications - establishing ground truth at multiple granularity levels.

2. **Public AI project postings:** AI-related project descriptions from public platforms (Upwork, Freelancer, Vietnamese tech Facebook groups) where sufficient technical follow-up documentation is publicly available or can be reconstructed.

3. **Expert interview data:** Semi-structured interviews with 5–8 experienced AI consultants/freelancers about their elicitation experiences, common failure patterns, and questions they always ask upfront. This data validates the taxonomy and surfaces patterns not present in the motivating case.

4. **Think-aloud session recordings:** Audio recordings and observer notes from sessions where non-technical participants interact with the elicitation assistant, capturing confusion points, IP-fear behaviors, and comprehension challenges.

#### Budget and Funding

The study is designed to be conducted within the resource constraints of a university research project with no external funding. Primary cost items are:

- LLM API costs for assistant development and evaluation: estimated at approximately $50–80 total for Claude API usage across development, testing, and evaluation phases (at current API pricing for the volume of calls required).
- Participant recruitment incentives for think-aloud sessions: nominal compensation (gift cards or equivalent) totaling approximately $200–300 for 8 participants.
- No specialized equipment, licensed datasets, or paid survey platforms are required.

The research's low resource requirement is itself a validity consideration: the approach must be practical for deployment in resource-constrained contexts such as university-affiliated AI startups or early-stage marketplace platforms.

#### Timeframe

The research is planned across 8 weeks, aligned with the academic semester sprint structure:

- **Week 1:** Literature synthesis, motivating case analysis, and concurrent expert interviews (n=3–5) for taxonomy development
- **Week 2:** Taxonomy validation, annotation rubric development, case sourcing, and ground truth annotation across 10–15 AI project cases
- **Week 3:** Elicitation assistant design, prompt engineering, implementation, and pilot testing on held-out cases
- **Week 4:** Quantitative evaluation of Conditions A (unaided developer) and B (rule-based checklist); concurrent think-aloud sessions with non-technical participants (n=5–8)
- **Week 5:** Quantitative evaluation of Conditions C (human structured interview) and D (LLM elicitation assistant); complete four-condition dataset assembled
- **Week 6:** Statistical analysis (ANOVA, post-hoc tests, effect sizes for H1–H4); thematic analysis of qualitative data; motivating case study reconstruction
- **Week 7:** Integration of quantitative and qualitative findings; drafting of all report sections
- **Week 8:** Internal review, revision, appendix finalization, and submission


#### Potential Challenges and Risks

**Challenge 1 - Ground truth annotation complexity.** Establishing expert-annotated ground truth for AI project requirements is inherently subjective. Mitigation: use of 3 independent annotators per case with inter-rater reliability measurement; structured annotation rubric derived from the taxonomy; iterative calibration sessions before formal annotation begins.

**Challenge 2 - Case diversity limitations.** The motivating case is from the AdTech domain; additional cases may cluster in similar domains, reducing generalizability. Mitigation: deliberate sampling strategy targeting diversity across AI problem types and industries; synthetic case construction for underrepresented categories if needed.

**Challenge 3 - LLM hallucination in elicitation output.** The assistant may generate plausible but incorrect clarifying questions that mislead rather than help. Mitigation: human expert review of assistant output in the evaluation phase; structured output schema that makes hallucinated or irrelevant questions identifiable by raters.

**Challenge 4 - Participant non-technical authenticity.** Participants in the think-aloud sessions may have more technical background than real-world non-technical founders, reducing ecological validity. Mitigation: screening questionnaire to verify no engineering or computer science educational background; use of proxy tasks (reading the motivating case's Facebook post and responding as the CEO) rather than simulating expertise they don't have.

**Challenge 5 - IP sensitivity in case collection.** Companies may be reluctant to share documented AI project cases due to confidentiality concerns. Mitigation: anonymization of all case materials before use; focus on publicly available postings for cases beyond the primary motivating case; synthetic case construction as a fallback for any domain where real cases cannot be obtained.

---

## 2. RESEARCH OBJECTIVES

1. To develop an AI-project-specific requirement dimension taxonomy that captures the unique requirement categories of AI/ML development projects absent from existing RE frameworks - including data availability, evaluation metrics, non-determinism, cost-accuracy trade-offs, LLMOps concerns, and expertise intersection requirements.

2. To design and implement an LLM-based requirement elicitation assistant that transforms vague, symptom-level AI project descriptions from non-technical clients into structured, AI-SDLC-aligned requirement specifications through multi-round interactive dialogue guided by the taxonomy.

3. To develop and validate the Requirement Coverage Score (RCS) metric as a standardized, reproducible measure of elicitation completeness for AI project requirements, enabling systematic comparison across elicitation conditions and generalizable to future AI project elicitation research.

4. To empirically evaluate the effectiveness of LLM-assisted elicitation compared to unaided, checklist-based, and human-interviewer approaches across multiple AI project cases, testing hypotheses H1–H4 regarding coverage, dual-persona identification, efficiency, and AI problem classification accuracy.

5. To identify and characterize the dual-persona elicitation problem - the systematic divergence between non-technical decision-maker descriptions and technical team knowledge - as a structural feature of AI outsourced project commissioning that must be accounted for in both elicitation methodology and AI marketplace platform design.

---

## 3. RESEARCH SCOPE

**In scope:**

- Requirements elicitation specifically for AI/ML outsourced service projects, where one party is a non-technical buyer and the other is an AI/ML domain expert or service provider.
- The upstream elicitation conversation phase: transforming vague initial problem descriptions into structured requirement sets. The research does not address downstream RE activities (requirements specification, formal modeling, verification) except as they define what "completeness" means for the elicitation output.
- The AI problem types most commonly encountered in the Vietnamese and regional AI outsourcing market: LLM application development, ML pipeline design, automation flows, data engineering, fine-tuning, and RAG system design. Highly specialized domains (robotics, embedded systems, computer vision at the hardware level) are out of scope for the primary study but may be addressed in future work.
- Evaluation in the context of the AITasker platform concept (SWP Topic 7): the elicitation assistant is designed as a feature within an AI-specialized marketplace, not as a standalone tool.
- English and Vietnamese language project descriptions, reflecting the bilingual nature of the Vietnamese AI services market.

**Out of scope:**

- Requirements engineering for traditional (non-AI) software projects.
- The downstream matching algorithm that uses elicitation output to pair clients with experts - this is a separate research problem addressed elsewhere in the AITasker platform design.
- Fine-tuning or retraining of LLM models for the elicitation task; the research uses prompt engineering and in-context learning with existing LLMs.
- Large-scale deployment or production implementation of the elicitation assistant; the research produces a validated prototype and design framework suitable for further development.
- Legal, ethical, or compliance requirement dimensions for AI systems (e.g., EU AI Act compliance), which constitute a separate and specialized requirement domain.

---

## 4. APPROACH AND METHOD

### 4.1 Overall Research Approach

The research adopts a Design Science Research (DSR) paradigm combined with empirical evaluation. DSR is appropriate because the primary research output is an artifact - the AI-project-specific taxonomy and elicitation assistant - evaluated against a practical problem. The empirical evaluation component uses a quasi-experimental mixed-methods design, with quantitative comparison across conditions and qualitative grounding through case study analysis and think-aloud sessions.

### 4.2 Phase 0 - AI-Project-Specific Requirement Dimension Taxonomy Development

**Technique:** Triangulated taxonomy development using three sources: (1) thematic analysis of the existing literature (Amershi et al. 2019, Sculley et al. 2015, Ahmad et al. 2023) to derive theoretically grounded dimensions; (2) case analysis of the motivating AdTech case to derive empirically grounded dimensions from a documented real-world elicitation failure; (3) semi-structured expert interviews with 5–8 AI consultants to validate and extend the taxonomy through practitioner knowledge.

**Dimensions to be validated:** Data availability and quality requirements; evaluation metric and ground truth definition requirements; AI problem type classification; cost-accuracy-speed constraint requirements; non-determinism handling requirements; structured I/O and observability design requirements; expertise intersection requirements (distinguishing ML, backend, data engineering, LLMOps skills); system context requirements (existing tech stack, current architecture); LLMOps operational requirements (drift detection, retraining, monitoring).

**Validation method:** Content validity index computed across expert raters; iterative revision until consensus; pilot application to 3 cases before formal use.

### 4.3 Phase 1 - Dataset Construction and Ground Truth Annotation

**Technique:** Purposive sampling of AI project cases with documented elicitation histories. Ground truth annotation using structured rubric derived from the taxonomy. Three independent annotators per case, with Cohen's kappa computed for inter-rater reliability (target ≥ 0.70). Disagreements resolved through structured discussion.

**Ground truth definition:** A requirement dimension is "surfaced" by an elicitation output if the output would lead a client to provide the specific information needed to fill that dimension in a complete AI project specification. Partial credit (0.5) is awarded if the dimension is partially addressed. Binary (0/1) scoring for each of the n taxonomy dimensions per case.

### 4.4 Phase 2 - Elicitation Assistant Design and Implementation

**Technique:** Prompt engineering using the Claude API (or equivalent LLM API), incorporating: (a) system prompt embedding the full AI-project-specific taxonomy as the assistant's knowledge structure; (b) few-shot examples drawn from documented AI project cases showing ideal elicitation question sequences; (c) multi-round dialogue structure where each client response unlocks progressively more specific follow-up questions; (d) structured output format that maps generated questions to taxonomy dimensions; (e) missing-dimension detection module that identifies which taxonomy categories remain uncovered after each exchange.

**Prompt design patterns applied:** Cognitive Verifier (assistant generates clarifying sub-questions), Question Refinement (assistant suggests better phrasings of vague client statements), Specification Disambiguation (assistant identifies and flags ambiguous statements with clarification requests), and Persona simulation (assistant adopts the perspective of a senior AI project consultant).

### 4.5 Phase 3 - Quantitative Experimental Evaluation

**Experimental design:** Between-subjects comparison across four conditions applied to the same set of 15–20 AI project cases. Each condition produces a requirement set for each case; requirement sets are evaluated against expert-annotated ground truth using the Requirement Coverage Score.

**Conditions:**
- Condition A (Baseline-0): Unaided developer - reads description, writes requirements without assistance.
- Condition B (Baseline-1): Rule-based checklist - applies a domain-agnostic RE checklist to the description.
- Condition C (Baseline-2): Human interviewer - conducts a structured interview using a standard template (no AI assistance).
- Condition D (Treatment): LLM elicitation assistant - multi-round interactive dialogue guided by the taxonomy.

**Primary metric - Requirement Coverage Score (RCS):**
RCS = (taxonomy dimensions surfaced / total taxonomy dimensions) × 100. Compared across conditions using ANOVA with post-hoc pairwise testing.

**Secondary metrics:** Elicitation time (minutes from receiving description to producing final requirement set); question relevance precision (proportion of generated questions that are relevant, evaluated by expert raters); client comprehensibility score (self-reported ease of answering questions, 1–5 Likert scale, from think-aloud participants); AI taxonomy classification accuracy (correct classification of AI problem type, binary per case).

### 4.6 Phase 4 - Qualitative Validation

**Technique 1 - Think-aloud sessions:** 5–8 non-technical participants interact with the elicitation assistant while narrating their thought process. Observer records: confusion points, questions that trigger defensive or evasive behavior, IP-fear signals, and points where the assistant successfully elicits information the participant would not have volunteered independently.

**Technique 2 - Expert interviews:** Semi-structured interviews with 5–8 AI consultants on their elicitation experiences. Thematic analysis using open coding to identify common failure patterns, successful strategies, and what information they consider essential to elicit upfront. Validates taxonomy completeness.

**Technique 3 - Case study reconstruction:** Detailed reconstruction of the motivating AdTech case, mapping at which point in the interaction each ground truth requirement dimension was first surfaced and what triggered the disclosure. Provides a longitudinal view of information emergence that quantitative metrics cannot capture.

---

## 5. RESEARCH PLAN

| No. | Date | Task | Expected Output | Person in Charge |
|-----|------|------|-----------------|------------------|
| 1 | Week 1 | Literature synthesis and case analysis of motivating AdTech case to derive taxonomy dimensions; concurrent expert interviews (n=3–5 AI consultants/practitioners) | Draft AI-project-specific requirement dimension taxonomy (v1) with 8–10 validated dimensions | All members |
| 2 | Week 2 | Expert validation and revision of taxonomy; annotation rubric development; case sourcing and selection (target 10–15 AI project cases from public postings, university networks, and synthetic construction) | Finalized taxonomy with inter-rater agreement ≥ 0.70; annotated ground truth requirement sets for all cases; case dataset ready for evaluation | All members |
| 3 | Week 3 | Elicitation assistant design and implementation: system prompt architecture, few-shot exemplar construction, multi-round dialogue structure, dimension-mapping output schema | Working elicitation assistant prototype; pilot test on 2–3 held-out cases; revised prompt architecture based on pilot findings | All members |
| 4 | Week 4 | Quantitative evaluation - Condition A (unaided developer) and Condition B (rule-based checklist) applied to full case set; parallel data collection for think-aloud sessions with non-technical participants (n=5–8) | Raw elicitation outputs for Conditions A and B; RCS scores computed; think-aloud session recordings and observer notes | All members |
| 5 | Week 5 | Quantitative evaluation - Condition C (human structured interview) and Condition D (LLM elicitation assistant) applied to full case set | Raw elicitation outputs for Conditions C and D; RCS scores computed; complete four-condition dataset ready for statistical analysis | All members |
| 6 | Week 6 | Statistical analysis (ANOVA, post-hoc pairwise tests, effect sizes for H1–H4); thematic analysis of think-aloud recordings and expert interview transcripts; motivating case study reconstruction mapping information emergence timeline | Statistical results table with hypothesis outcomes (H1–H4 accepted/rejected); thematic codebook from qualitative data; case study narrative | Minh Hùng |
| 7 | Week 7 | Integration of quantitative and qualitative findings; identification of design implications for AITasker platform; drafting of all report sections | Complete integrated findings; draft research report including all sections (Introduction, Methodology, Results, Discussion, Conclusion) | All members |
| 8 | Week 8 | Internal review and revision; finalization of appendices (taxonomy, prompt architecture, annotation rubric, statistical protocol); submission | Final submitted research report and all appendices | All members |

---

## 6. EXPECTED RESULTS

### Theoretical Implications

This research contributes a theoretically grounded AI-project-specific requirement dimension taxonomy that extends requirements engineering theory into a domain that existing RE frameworks do not adequately address. The taxonomy formalizes AI-project concerns (data, evaluation, non-determinism, operational monitoring, expertise intersection) as first-class requirement dimensions, establishing a conceptual foundation that future RE research can build on. The identification and operationalization of the dual-persona elicitation problem contributes a new theoretical construct to RE research - one that applies beyond AI projects to any outsourcing context where non-technical buyers commission from highly specialized experts.

### Methodological Implications

The Requirement Coverage Score (RCS) metric provides the first standardized, reproducible measurement approach for elicitation completeness in the AI project domain. Unlike generic completeness metrics, RCS is calibrated against a domain-specific taxonomy, making it sensitive to AI-project-specific omissions that general metrics would miss. The metric and its associated annotation protocol can be adopted by future researchers studying elicitation for other specialized technical domains. The experimental design - comparing LLM-assisted against unaided, checklist, and human-interviewer baselines across documented AI project cases with expert-annotated ground truth - provides a methodological template for LLM4RE evaluation research that addresses the ecological validity gap identified by Zadenoori et al. (2025).

### Practical Implications

The primary practical output is a validated prototype elicitation assistant and its underlying prompt architecture, which can be directly embedded into the AITasker platform (SWP Topic 7) as the requirement elicitation feature for the client side of the marketplace. The taxonomy informs the platform's requirement capture forms, expert-matching criteria, and scope-change detection mechanisms. More broadly, the findings on which taxonomy dimensions are most frequently missed by non-technical buyers and which are most effectively surfaced by the LLM assistant inform where human expert intervention is most valuable in the elicitation process - enabling a human-in-the-loop design that allocates expert review effort to the highest-risk requirement gaps.

The research also produces practical design guidelines for AI marketplace platforms addressing: how to structure the pre-contract elicitation conversation to protect both buyer IP and expert solution design; how to detect dual-persona information gaps and prompt for technical team involvement; and how to classify AI problem types from vague descriptions to improve matching precision.

### Plans for Practical Application

The validated elicitation assistant and taxonomy will be submitted to the AITasker platform development team (SWP Topic 7) as a functional specification for the platform's requirement elicitation module. The RCS metric and annotation protocol will be published as open resources for the LLM4RE research community. The dual-persona elicitation framework will be submitted for presentation at a relevant software engineering or AI conference(if possible). The research findings on AI project elicitation failure patterns have direct application in training materials for AI consultants entering the Vietnamese AI services market.

---

## REFERENCES

[1] Zowghi, D., & Coulin, C. (2005). Requirements elicitation: A survey of techniques, approaches, and tools. In A. Aurum & C. Wohlin (Eds.), *Engineering and managing software requirements* (pp. 19–46). Springer.

[2] Nuseibeh, B., & Easterbrook, S. (2000). Requirements engineering: A roadmap. In *Proceedings of the conference on the future of software engineering* (pp. 35–46). ACM/IEEE.

[3] Goguen, J. A., & Linde, C. (1993). Techniques for requirements elicitation. In *Proceedings of the IEEE International Symposium on Requirements Engineering* (pp. 152–164). IEEE.

[4] Zadenoori, M. A., Dabrowski, J., Alhoshan, W., Zhao, L., & Ferrari, A. (2025). Large language models (LLMs) for requirements engineering (RE): A systematic literature review. *arXiv preprint arXiv:2509.11446*.

[5] Hemmat, A., Sharbaf, M., Kolahdouz-Rahimi, S., Lano, K., & Tehrani, S. Y. (2025). Research directions for using LLM in software requirement engineering: A systematic review. *Frontiers in Computer Science, 7*, 1519437. https://doi.org/10.3389/fcomp.2025.1519437

[6] Arora, C., Grundy, J., & Abdelrazek, M. (2023). Advancing requirements engineering through generative AI: Assessing the role of LLMs. In *Generative AI for Effective Software Development* (pp. 129–148). Springer. https://arxiv.org/pdf/2310.13976

[7] Ferrari, A., Spoletini, P., & Gnesi, S. (2017). Natural language processing for requirements engineering: The best is yet to come. In *Proceedings of the IEEE 25th International Requirements Engineering Conference* (pp. 429–432). IEEE.

[8] Luitel, D., Hassani, S., & Sabetzadeh, M. (2024). Improving requirements completeness: Automated assistance through large language models. *Requirements Engineering, 29*(1), 73–95. https://doi.org/10.1007/s00766-024-00416-3

[9] Amershi, S., Begel, A., Bird, C., DeLine, R., Gall, H., Kamar, E., Nagappan, N., Nushi, B., & Zimmermann, T. (2019). Software engineering for machine learning: A case study. In *Proceedings of the 41st International Conference on Software Engineering: SEIP* (pp. 291–300). IEEE.

[10] Sculley, D., Holt, G., Golovin, D., Davydov, E., Phillips, T., Ebner, D., Chaudhary, V., Young, M., Crespo, J.-F., & Dennison, D. (2015). Hidden technical debt in machine learning systems. In *Advances in Neural Information Processing Systems* (Vol. 28, pp. 2503–2511). NeurIPS.

[11] Fan, A., Gokkaya, B., Harman, M., Lyubarskiy, M., Sengupta, S., Yoo, S., & Zhang, J. M. (2023). Large language models for software engineering: Survey and open problems. In *2023 IEEE/ACM International Conference on Software Engineering: Future of Software Engineering* (pp. 31–53). IEEE.

[12] White, J., Hays, S., Fu, Q., Spencer-Smith, J., & Schmidt, D. C. (2024). ChatGPT prompt patterns for improving code quality, refactoring, requirements elicitation, and software design. In *Generative AI for Effective Software Development* (pp. 71–108). Springer.

[13] Vogelsang, A., & Fischbach, J. (2024). Using large language models for natural language processing tasks in requirements engineering: A systematic guideline. *arXiv preprint arXiv:2402.13823*.

[14] Cheng, H., Husen, J. H., Peralta, S. R., Jiang, B., Yoshioka, N., Ubayashi, N., et al. (2024). Generative AI for requirements engineering: A systematic literature review. *arXiv preprint arXiv:2409.06741*.

[15] Ahmad, K., Abdelrazek, M., Arora, C., Bano, M., & Grundy, J. (2023). Requirements engineering framework for human-centered artificial intelligence software systems. *Applied Soft Computing, 143*, 110455.

[16] Zhao, L., Alhoshan, W., Ferrari, A., Letsholo, K. J., Ajagbe, M. A., Chioasca, E.-V., & Batista-Navarro, R. T. (2021). Natural language processing for requirements engineering: A systematic mapping study. *ACM Computing Surveys, 54*(3), 1–41.

[17] Mich, L., Franch, M., & Novi Inverardi, P. (2004). Market research for requirements analysis using linguistic tools. *Requirements Engineering, 9*(1), 40–56.

---

## APPENDICES

**Appendix A - AI-Project-Specific Requirement Dimension Taxonomy (Draft v1)**
To be completed during Phase 0 of the research. Will include all dimension definitions, sub-dimensions, example elicitation questions, and scoring criteria for the Requirement Coverage Score.

**Appendix B - Elicitation Assistant System Prompt Architecture**
Full prompt design including system prompt text, few-shot example structure, output schema for dimension-mapped questions, and missing-dimension detection logic. To be completed during Phase 3.

**Appendix C - Ground Truth Annotation Rubric**
Detailed scoring instructions for expert annotators, including dimension definitions, partial credit criteria, disagreement resolution protocol, and inter-rater reliability calculation method.

**Appendix D - Case Dataset Summary**
Anonymized summary of the 15–20 AI project cases used in the evaluation, including AI problem type, industry, initial description source, and ground truth dimension coverage profile.

**Appendix E - Statistical Analysis Protocol**
Detailed specification of all statistical tests to be applied, including ANOVA assumptions testing, post-hoc comparison method (Tukey HSD), effect size measures (Cohen's d), and significance thresholds.