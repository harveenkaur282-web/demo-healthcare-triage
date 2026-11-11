one analyzes symptoms, one classifies urgency and one books an appointment 
calling each agent in order when patient complaint comes in

the tool does keyword matching against the dictionary- symptoms
the llm does not invent any medical knowledge
Fresh agents instances per request to avoid context speed. This is the pattern.
It's a plain Python function that calls each agent in sequence

Healthcare Triage Architecture
The healthcare triage system relies on a Coordinator pattern. Instead of using one massive agent to handle everything, the architecture breaks the workflow down into three specialized agents:

SymptomAnalyzer: Maps the raw complaint to specific conditions and severities using a lookup_symptoms tool.
UrgencyClassifier: Evaluates the severities to assign a priority level (urgent, standard, or routine) using a classify_urgency tool.
AppointmentScheduler: Uses the assigned priority to book an available time slot using a book_appointment tool.
The coordinator calls each agent in sequence, passing the parsed output of one agent as input to the next. The full implementation in healthcare_triage.py includes setup boilerplate (model IDs, AWS region), a sample SYMPTOM_CONDITIONS knowledge base, an AVAILABLE_SLOTS table, and helpers like clean_response and run_agent_with_retry — those are not the focus of this lesson.

The key pattern: capture tool results in shared state
Before defining the agents, the demo establishes a _tool_results dictionary. Each tool writes its raw JSON output into this dictionary so the coordinator can read it directly, instead of trying to parse the LLM's natural-language response.

# Shared dict for capturing tool results directly.
# LLMs may rephrase or modify tool output in their text response,
# so we capture the raw tool result here for reliable downstream use.
_tool_results = {}
Inside each tool the last line writes _tool_results[<key>] = result before returning. That is the contract the coordinator depends on.

One agent builder, three repetitions
All three agents are built with the same shape: a BedrockModel at temperature=0.0, a @tool-decorated function that performs a lookup and writes into _tool_results, and a system prompt that scopes the agent to a single responsibility. Here is the SymptomAnalyzer:

def build_symptom_analyzer() -> Agent:
    model = BedrockModel(model_id=MODEL_ID, region_name=AWS_REGION, temperature=0.0)

    @tool
    def lookup_symptoms(complaint_text: str) -> str:
        # ... match complaint_text against SYMPTOM_CONDITIONS keywords
        result = json.dumps({"matched_symptoms": matches, ...})
        _tool_results["symptoms"] = result  # capture for coordinator
        return result

    system_prompt = """You are a Symptom Analyzer agent. Your ONLY job is symptom analysis.
Call the lookup_symptoms tool with the patient's complaint text.
After the tool returns, output ONLY the raw JSON result. Do not classify urgency or book appointments."""

    return Agent(model=model, system_prompt=system_prompt, tools=[lookup_symptoms])
build_urgency_classifier and build_appointment_scheduler follow the same builder pattern — different tool, different _tool_results key ("urgency" and "booking"), and a system prompt that forbids the agent from doing anything outside its single job.

Orchestrating the pipeline
The coordinator wires the three agents together. It clears _tool_results for every patient (so one patient's data does not bleed into the next), instantiates a fresh agent at each stage, and threads the JSON output of one stage into the prompt of the next.

def run_triage_pipeline(patient: dict) -> dict:
    _tool_results.clear()  # Reset between patients to avoid stale data

    run_agent_with_retry(build_symptom_analyzer,
        f"Analyze these symptoms: {patient['complaint']}")
    symptom_json = json.loads(_tool_results["symptoms"])

    run_agent_with_retry(build_urgency_classifier,
        f"Classify urgency for this symptom analysis: {json.dumps(symptom_json)}")
    urgency_level = json.loads(_tool_results["urgency"])["urgency"]

    run_agent_with_retry(build_appointment_scheduler,
        f"Book an appointment for patient_id={patient['patient_id']} with urgency_level={urgency_level}")
    return json.loads(_tool_results["booking"])
The coordinator is the only component that knows about the overall workflow — the agents themselves remain narrowly scoped. Note that it never parses LLM prose; it always reads the structured JSON from _tool_results.


# QUES: Why does the system instantiate a fresh agent for every patient rather than reusing the same agent instance?
TO PREVENT THE CONTEXT AND HISTORY OF ONE PATIENT FROM BLEEDING INTO THE NEXT PATIENT'S PROCESSING 


