# Multi-Account GenAI Governance & Data Residency on AWS Bedrock

## 1. Context
As enterprise organisations accelerate the adoption of Generative AI platforms such as Amazon Bedrock, cloud platform teams must ensure that AI workloads adhere strictly to enterprise data boundaries and regulatory compliance frameworks—most notably **UK GDPR (Chapter V – International Transfers)** and the **EU AI Act**. In a modern multi-account AWS Organization governed by AWS Control Tower, foundational guardrails must be established at the landing zone perimeter so development teams can consume foundation models rapidly without introducing data sovereignty risks.

---

## 2. Problem Statement
Default configurations and modern features in Amazon Bedrock introduce significant data residency and security exposure across multi-account landing zones:
1. **Dynamic Cross-Region Routing:** Amazon Bedrock provides **Global System Inference Profiles (`global.*`)** designed to dynamically distribute prompt traffic across global regions (including the US and APAC) to optimize throughput. Dynamic routing outside the UK/EEA without formal transfer safeguards constitutes an unauthorized international data transfer.
2. **Third-Party Model & Licensing Sprawl:** Foundation models from third-party vendors (such as Anthropic, Cohere, and AI21) are procured through the AWS Marketplace. Workload teams with administrator or power-user IAM permissions could subscribe to unvetted vendor models and accept commercial/data terms without enterprise security or legal review.
3. **Control Tower Preventive Guardrail Interoperability:** AWS Control Tower's regional boundary guardrails (`GRREGIONDENY`) flag system inference profile invocations as having an `unspecified` region, unintentionally blocking legitimate European inference profiles unless specifically accounted for.

---

## 3. Objective
* Architect and evaluate a multi-layered defense-in-depth governance model across landing zone environments.
* Centrally block all unapproved dynamic cross-border prompt routing (e.g., `global.*` inference profiles).
* Restrict foundation model access to an approved enterprise catalog using declarative Service Control Policies (SCPs).
* Gate third-party AWS Marketplace model subscriptions so account operators cannot self-subscribe to unapproved vendor EULAs.
* Provide workload developers with compliant architectural patterns (such as EU-scoped system and application inference profiles) without creating code-level friction.

---

## 4. Tools & Technology Used
* **Cloud Platform:** AWS Organizations, Amazon Bedrock, AWS Service Control Policies (SCPs), AWS Control Tower, AWS IAM, AWS Marketplace.
* **Infrastructure as Code & Automation:** Terraform, Customisations for Control Tower (CfCT), AWS CodePipeline, Git.
* **Governance, Observability & Security:** Amazon Bedrock Guardrails, AWS CloudTrail, AWS CloudWatch.
* **Compliance Frameworks:** UK GDPR (Chapter V: International Data Transfers), EU AI Act, Information Commissioner's Office (ICO) Guidance.

---

## 5. My Contribution
* **Architectural Spike & Research:** Led the technical investigation into Amazon Bedrock inference profiles, evaluating trade-offs across direct regional invocation, EU cross-region inference, global routing, Bedrock Guardrails, and centralized LLM gateways.
* **5-Layer Defense-in-Depth Design:** Designed a cohesive governance model spanning landing zone regional perimeters, inference profile blocks, model allowlists, Marketplace gates, and payload sanitisation.
* **Policy Prototyping & Authoring:**
  * Authored SCP statements utilising the `NotResource` allowlist pattern to restrict direct foundation model invocations to approved model versions and `eu.*` inference profiles.
  * Designed conditional SCP statements targeting `aws-marketplace:ProductId` on `aws-marketplace:Subscribe` to prevent rogue vendor subscriptions.
  * Engineered condition-key solutions (`bedrock:InferenceProfileArn`) to account for Control Tower's `aws:RequestedRegion: unspecified` platform behavior.
* **Developer Experience & Tooling Guidance:** Formulated AWS CLI and SDK patterns enabling application teams to discover EU-confined system profiles (`eu.*`) and provision custom Application Inference Profiles for project tagging and cost tracking.
* **Stakeholder Engagement & Backlog Preparation:** Authored the comprehensive RFC/Spike findings paper on Confluence, engaged AWS Enterprise Support to clarify native catalog capabilities, obtained peer review and security sign-off, and structured follow-up delivery stories for Three Amigos refinement.

---

## 6. Testing Approach
* **Negative Verification (Deny Enforcement):**
  * Tested invocations using unapproved foundation model ARNs (e.g., unvetted models or non-approved versions) to confirm they fail with explicit `AccessDeniedException` responses.
  * Attempted invocation of global inference profiles (`arn:aws:bedrock:*:*:inference-profile/global.*`) to verify traffic is rejected at the SCP perimeter.
  * Attempted subscription actions against unapproved third-party model Marketplace listings to verify policy rejection based on the `aws-marketplace:ProductId` condition key.
* **Positive Verification (Allow Compliance):**
  * Executed invocations against approved models (e.g., approved Claude 3.5 Sonnet releases, Amazon Titan) and verified successful responses.
  * Provisioned and tested an EU-scoped Application Inference Profile (`eu.*` source) to verify dynamic European failover without triggering `GRREGIONDENY` blocks.
* **Platform Interaction Verification:**
  * Analyzed AWS CloudTrail event payloads to inspect `eventSource`, `aws:RequestedRegion`, and request parameters during inference profile routing to confirm boundary compliance.

---

## 7. Outcome
* **Zero Cross-Border Data Leakage:** Established an organizational baseline that eliminates the risk of prompt payloads crossing outside approved UK/EEA borders.
* **Centralized Model Catalog Governance:** Replaced open model consumption with an auditable, declarative allowlist mechanism governed at the landing zone layer.
* **Seamless Developer Experience:** Application teams maintain high availability through EU cross-region inference without requiring code changes beyond specifying the `eu.` model prefix.
* **Successful Verification:** Technical approach, documentation, and policy configurations successfully passed peer review and security verification, progressing directly into production implementation stories.

---

## 8. Key Learnings
1. **Control Tower & Region Evaluation Quirks:** When calling AWS system-defined inference profiles, the AWS request engine sets `aws:RequestedRegion` to `"unspecified"`. Standard region deny guardrails (`GRREGIONDENY`) will intercept these requests unless targeted condition checks (e.g., `bedrock:InferenceProfileArn`) or specific exceptions are implemented.
2. **Marketplace Models vs. Direct Models:** Foundation models sold via AWS Marketplace enforce access via subscription actions (`aws-marketplace:Subscribe` requiring Product IDs). However, models offered natively by AWS (e.g., Amazon Titan) or non-Marketplace partners lack Marketplace Product IDs and must be governed strictly through `bedrock:InvokeModel*` resource ARNs in SCPs.
3. **Content Guardrails ≠ Network Routing:** Amazon Bedrock Guardrails are vital for content sanitation and redacting PII (e.g., National Insurance numbers, banking details), but they provide zero network or routing boundaries. Compliance requires pairing content guardrails with infrastructure-level routing SCPs.

---

## 9. References
* [AWS Bedrock Documentation: Cross-Region Inference](https://docs.aws.amazon.com/bedrock/latest/userguide/cross-region-inference.html)
* [AWS Bedrock Documentation: Identity-Based Policy Examples](https://docs.aws.amazon.com/bedrock/latest/userguide/security_iam_id-based-policy-examples.html)
* [UK Information Commissioner's Office (ICO): International Transfers (UK GDPR Chapter V)](https://ico.org.uk/for-organisations/uk-gdpr-guidance-and-resources/international-transfers/)
* [AWS Control Tower Documentation: Guardrail Reference and Regional Restrictions](https://docs.aws.amazon.com/controltower/latest/userguide/guardrails-reference.html)