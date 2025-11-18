Comprehensive Guide: Adding Anthropic API Support
Overview
This guide will help you add Anthropic Claude models (Sonnet 4.5, Sonnet 4.0, Haiku 4.5) to your TFDA system while maintaining all existing features.
Modified Code Sections
1. Import Section (Add after line 15)
pythonfrom anthropic import Anthropic
Location: Add this import after from xai_sdk.chat import user as xai_user, system as xai_system

2. ModelChoice Dictionary (Replace lines 70-78)
pythonModelChoice = { 
    "gpt-5-nano": "openai", 
    "gpt-4o-mini": "openai", 
    "gpt-4.1-mini": "openai", 
    "gemini-2.5-flash": "gemini", 
    "gemini-2.5-flash-lite": "gemini", 
    "grok-4-fast-reasoning": "grok", 
    "grok-3-mini": "grok",
    "claude-sonnet-4.5": "anthropic",
    "claude-sonnet-4-20250514": "anthropic",
    "claude-haiku-4.5": "anthropic",
}
Location: Replace the existing ModelChoice dictionary (lines 70-78)

3. LLMRouter.init Method (Replace lines 80-82)
pythondef __init__(self): 
    self._openai_client = None 
    self._gemini_ready = False 
    self._xai_client = None 
    self._anthropic_client = None
    self._init_clients()
Location: Replace the __init__ method in the LLMRouter class

4. LLMRouter._init_clients Method (Replace lines 84-91)
pythondef _init_clients(self): 
    if os.getenv("OPENAI_API_KEY"): 
        self._openai_client = OpenAI(api_key=os.getenv("OPENAI_API_KEY")) 
    if os.getenv("GEMINI_API_KEY"): 
        genai.configure(api_key=os.getenv("GEMINI_API_KEY")) 
        self._gemini_ready = True 
    if os.getenv("XAI_API_KEY"): 
        self._xai_client = XAIClient(api_key=os.getenv("XAI_API_KEY"), timeout=3600)
    if os.getenv("ANTHROPIC_API_KEY"):
        self._anthropic_client = Anthropic(api_key=os.getenv("ANTHROPIC_API_KEY"))
Location: Replace the _init_clients method in the LLMRouter class

5. LLMRouter.generate_text Method (Replace lines 93-100)
pythondef generate_text(self, model_name: str, messages: List[Dict], params: Dict) -> Tuple[str, Dict, str]: 
    provider = ModelChoice.get(model_name, "openai") 
    if provider == "openai": 
        return self._openai_chat(model_name, messages, params), {"total_tokens": self._estimate_tokens(messages)}, "OpenAI" 
    elif provider == "gemini": 
        return self._gemini_chat(model_name, messages, params), {"total_tokens": self._estimate_tokens(messages)}, "Gemini" 
    elif provider == "grok": 
        return self._grok_chat(model_name, messages, params), {"total_tokens": self._estimate_tokens(messages)}, "Grok"
    elif provider == "anthropic":
        return self._anthropic_chat(model_name, messages, params), {"total_tokens": self._estimate_tokens(messages)}, "Anthropic"
Location: Replace the generate_text method in the LLMRouter class

6. LLMRouter.generate_vision Method (Replace lines 102-109)
pythondef generate_vision(self, model_name: str, prompt: str, images: List) -> str: 
    provider = ModelChoice.get(model_name, "openai") 
    if provider == "gemini": 
        return self._gemini_vision(model_name, prompt, images) 
    elif provider == "openai": 
        return self._openai_vision(model_name, prompt, images)
    elif provider == "anthropic":
        return self._anthropic_vision(model_name, prompt, images)
    return "Vision not supported"
Location: Replace the generate_vision method in the LLMRouter class

7. Add New Anthropic Methods (Insert after line 149)
pythondef _anthropic_chat(self, model: str, messages: List, params: Dict) -> str:
    # Convert messages to Anthropic format
    system_msgs = [m["content"] for m in messages if m["role"] == "system"]
    system_prompt = "\n\n".join(system_msgs) if system_msgs else ""
    
    anthropic_messages = []
    for m in messages:
        if m["role"] == "user":
            anthropic_messages.append({"role": "user", "content": m["content"]})
        elif m["role"] == "assistant":
            anthropic_messages.append({"role": "assistant", "content": m["content"]})
    
    # If no user messages, add the system content as user message
    if not anthropic_messages:
        anthropic_messages.append({"role": "user", "content": system_prompt})
        system_prompt = ""
    
    kwargs = {
        "model": model,
        "messages": anthropic_messages,
        "temperature": params.get("temperature", 0.4),
        "top_p": params.get("top_p", 0.95),
        "max_tokens": params.get("max_tokens", 800)
    }
    
    if system_prompt:
        kwargs["system"] = system_prompt
    
    response = self._anthropic_client.messages.create(**kwargs)
    return response.content[0].text

def _anthropic_vision(self, model: str, prompt: str, images: List) -> str:
    content = [{"type": "text", "text": prompt}]
    
    for img in images:
        buf = io.BytesIO()
        img.save(buf, format="PNG")
        b64 = base64.b64encode(buf.getvalue()).decode("utf-8")
        content.append({
            "type": "image",
            "source": {
                "type": "base64",
                "media_type": "image/png",
                "data": b64
            }
        })
    
    response = self._anthropic_client.messages.create(
        model=model,
        messages=[{"role": "user", "content": content}],
        max_tokens=1024
    )
    return response.content[0].text
Location: Insert these methods after the _estimate_tokens method (around line 149)

8. Sidebar Provider Status (Insert after line 443)
pythonshow_provider_status("Anthropic", "ANTHROPIC_API_KEY")
Location: Add this line after the existing show_provider_status calls in the sidebar section (after line 443)

9. Update Active Providers Counter (Replace lines 467-471)
pythonproviders_ok = sum([ 
    bool(os.getenv("OPENAI_API_KEY")), 
    bool(os.getenv("GEMINI_API_KEY")), 
    bool(os.getenv("XAI_API_KEY")),
    bool(os.getenv("ANTHROPIC_API_KEY"))
])
st.markdown(f""" 
    <div class="wow-card"> 
        <div class="metric-value">{providers_ok}/4</div> 
        <div class="metric-label">Active Providers</div> 
    </div> 
    """, unsafe_allow_html=True)
Location: Replace the providers counter section in the header

10. Update Model Selection Dropdowns (Multiple locations)
Location 1: Line 513 (OCR section)
pythonllm_ocr_model = st.selectbox("LLM Model", [ 
    "gemini-2.5-flash", 
    "gemini-2.5-flash-lite", 
    "gpt-4o-mini",
    "claude-sonnet-4.5",
    "claude-haiku-4.5"
])
Location 2: Line 572 (Agent Config section)
pythonagent["model"] = st.selectbox( 
    "Model", 
    ["gpt-4o-mini", "gpt-5-nano", "gemini-2.5-flash", "gemini-2.5-flash-lite", 
     "grok-3-mini", "claude-sonnet-4.5", "claude-sonnet-4-20250514", "claude-haiku-4.5"], 
    index=0, 
    key=f"model_{i}" 
)

11. Update Chart Color Mapping (Multiple locations)
Location 1: Line 698 (Latency chart)
pythoncolor_discrete_map={
    "OpenAI": "#10a37f", 
    "Gemini": "#4285f4", 
    "Grok": "#ff6b6b",
    "Anthropic": "#d97757"
}
Location 2: Line 713 (Token usage chart) - Same as above
Location 3: Line 729 (Provider pie chart) - Same as above
Location: Update all three color_discrete_map dictionaries in the Dashboard tab

12. Update DEFAULT_FDA_AGENTS YAML (Optional - for one agent as example)
Find the first agent in DEFAULT_FDA_AGENTS (around line 185) and change its model:
yaml  - name: 申請資料提取器 
    description: 進行繁體中文摘要 
    system_prompt: | 
      你是一位醫療器材法規專家。根據提供的文件，進行繁體中文摘要in markdown in traditional chinese with keywords in coral color. Please also create a table include 20 key items。
      - 識別：廠商名稱、地址、品名、類別、證書編號、日期、機構 
      - 標註不確定項目，保留原文引用 
      - 以結構化格式輸出（表格或JSON） 
    user_prompt: "你是一位醫療器材法規專家。根據提供的文件，進行繁體中文摘要in markdown in traditional chinese with keywords in coral color. Please also create a table include 20 key items。" 
    model: claude-sonnet-4.5
    temperature: 0 
    top_p: 0.9 
    max_tokens: 6000

13. Update Footer Text (Line 790)
pythonst.markdown(f"""<div style="text-align: center; padding: 2rem; opacity: 0.7;"> 
    <p>{theme_icon} <strong>TFDA Agentic AI Assistance Review System</strong></p> 
    <p>Powered by OpenAI, Google Gemini, xAI Grok & Anthropic Claude • Built with Streamlit</p> 
    <p style="font-size: 0.8rem;">© 2024 • Theme: {st.session_state.theme}</p></div>""", unsafe_allow_html=True)
Location: Replace the footer text at the end of the file
