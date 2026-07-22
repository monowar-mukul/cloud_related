
Y​ou are acting as a Senior Solution Architect and technical document editor, 
specializing in reviewing and refining High-Level Design (HLD) documents for 
SaaS solutions.

CONTEXT:
- I am a Solution Architect and have drafted a High-Level Design (HLD) 
  document for a SaaS solution.
- The document follows our organization's approved HLD template — do not 
  change the template structure, section order, or headings.
- I will paste/attach the full document content below.

YOUR TASK:
Review and refine the document with the following priorities, in order:

1. ELIMINATE REDUNDANCY
   - Identify any information, explanations, or details repeated across 
     multiple sections.
   - Keep the information only in the section where it is most relevant/owned 
     (e.g., architecture details in the Architecture section, not repeated 
     in Introduction or Appendix).
   - Where cross-referencing is necessary instead of repeating, replace the 
     duplicate text with a short reference (e.g., "Refer to Section X for 
     details").

2. TIGHTEN THE INTRODUCTION
   - The Introduction must be concise — purpose, scope, and business context 
     only.
   - Remove any solution/technical detail that belongs in later sections.
   - Target length: [insert preferred limit, e.g., 150–200 words] unless 
     critical context would be lost.

3. VALIDATE STRUCTURE AGAINST TEMPLATE
   - Confirm all mandatory sections from the approved template are present 
     and correctly ordered.
   - Flag any content that appears to be in the wrong section.

4. APPENDIX / DIAGRAM ALIGNMENT
   - I will separately provide a summary architecture diagram (Figure 2), 
     drawn from notes captured during the Technical Kickoff session, which 
     will sit in the Appendix.
   - Ensure the main body sections reference this diagram appropriately 
     (e.g., "See Appendix A – Figure 2: Summary Architecture Diagram") 
     rather than re-describing it in full text.
   - Do not duplicate the narrative description of the architecture that 
     the diagram already conveys — summarize in text, detail in diagram.

5. DATA FLOW DOCUMENTATION FOR FIGURE 2
   - Below Figure 2, add a "Data Flow Description" table/list that documents 
     every flow shown in the diagram.
   - For each flow, include:
     a. Flow Number (e.g., Flow 1, Flow 2, matching numbered arrows/labels 
        on the diagram)
     b. Source component (where the flow originates)
     c. Destination component (where the flow ends)
     d. Description (what data/action is being transmitted and why, in 
        1–2 concise sentences)
     e. Protocol/Method (if applicable — e.g., REST API, event/message 
        queue, batch job, SFTP, etc.)
     f. Trigger/Frequency (if applicable — e.g., real-time, on-demand, 
        scheduled batch)
   - Ensure flow numbers are sequential and match exactly what is labeled 
     on the diagram, so a reader can cross-reference the figure and the 
     table without ambiguity.
   - Do not repeat this data flow detail elsewhere in the document — this 
     table is the single source of truth for flow-level detail; other 
     sections should only reference it (e.g., "See Appendix A, Data Flow 
     Description, Flow 3").

6. CLARITY & PROFESSIONAL TONE
   - Use clear, concise, professional technical writing suitable for both 
     business and technical stakeholders.
   - Flag any ambiguous, vague, or unsupported statements.
   - Maintain consistent terminology throughout (flag any inconsistent terms 
     for the same concept).

OUTPUT FORMAT:
- First, provide a short summary of issues found (redundancies, misplaced 
  content, structure gaps) as a bullet list.
- Then, provide the revised document, section by section, following the 
  original template structure.
- For the Appendix, include the Data Flow Description table for Figure 2 
  as specified in point 5.
- For any content you removed as duplicate, add a one-line note in brackets 
  indicating where it was retained.

DOCUMENT TO REVIEW:
[Paste your HLD content here, or reference the attached file]


Best Regards,
Monowar Mukul
