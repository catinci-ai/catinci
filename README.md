# Catinci

Catinci is a screen-free, voice-first AI assistant designed to support kids’ curiosity
while giving parents tools to guide learning safely.

## What Catinci Does
When a child asks a question, CatInci:
1. Answers via a voice agent
2. Allows parents to verify the answer (Fact Check)
3. Sends kid-friendly videos/images (Show Me)
4. Recommends local library books (in progress)

## Repositories
- fact_check
  Voice-activated verification using reputable sources
- show_me
  Image and video discovery for parent follow-up

## Tech Stack (high level)
- ElevenLabs voice agents
- Custom client tools
- Web app frontend

## Step-by-Step Setup
1. **Install prerequisites**
   - Python 3.10+ and `pip`.
   - Twilio SMS-capable account and phone number (for Show Me).
   - Google Programmable Search Engine (CSE) with API key + CX ID, and Gemini API key (for Fact Check searches).
   - (Optional) Render or similar hosting if you want to deploy beyond local testing.

2. **Unpack the repos**
   ```bash
   cd /path/to/your/workspace
   unzip catinci-showme-main.zip
   unzip catinci-factcheck-main.zip
   ```
   Keep the three folders (`catinci-main`, `catinci-showme-main`, `catinci-factcheck-main`) at the same directory level so the documentation stays in sync.

3. **Configure environment variables**
   - `catinci-showme-main/.env`
     ```
     TWILIO_SID=ACxxxxxxxxxxxxxxxxxxxxxxxxxxxxxx
     TWILIO_TOKEN=yyyyyyyyyyyyyyyyyyyyyyyyyyyyyy
     TWILIO_FROM=+15555555555
     ```
   - `catinci-factcheck-main/.env`
     ```
     GOOGLE_API_KEY=zzzzzzzzzzzzzzzzzzzzzzzzzzzz
     GOOGLE_CX_ID=your_cse_id
     GEMINI_API_KEY=aaaaaaaaaaaaaaaaaaaaaaaaaaaa
     ```
   Store `.env` files locally (never commit them) and load them with `source` or use a process manager that injects env vars.

4. **Create and activate virtual environments**
   ```bash
   # Show Me service
   cd catinci-showme-main
   python3 -m venv venv
   source venv/bin/activate
   pip install -r requirements.txt

   # Fact Check service (in a new shell tab)
   cd ../catinci-factcheck-main
   python3 -m venv venv
   source venv/bin/activate
   pip install -r requirements.txt
   ```
   Run each service in its own shell tab to keep environments isolated.

5. **Run the services locally**
   - Show Me (Twilio SMS helper):
     ```bash
     cd catinci-showme-main
     source venv/bin/activate
     python show_me_server.py  # listens on PORT or 5002
     ```
   - Fact Check (claim verifier):
     ```bash
     cd catinci-factcheck-main
     source venv/bin/activate
     python server.py  # listens on PORT or 5001
     ```
   Confirm health endpoints:
   ```bash
   curl http://127.0.0.1:5002/health
   curl http://127.0.0.1:5001/health
   ```

6. **Smoke-test the APIs**
   - Store a Show Me topic, then trigger the SMS send:
     ```bash
     curl -X POST http://127.0.0.1:5002/show_me \
       -H "Content-Type: application/json" \
       -d '{"text":"I want to see volcano pictures","parent_phone":"+15551234567"}'
     curl -X POST http://127.0.0.1:5002/show_me \
       -H "Content-Type: application/json" \
       -d '{"text":"show me","parent_phone":"+15551234567"}'
     ```
   - Fact Check baseline and intent-aware flow:
     ```bash
     curl -X POST http://127.0.0.1:5001/fact_check \
       -H "Content-Type: application/json" \
       -d '{"claim":"Penguins live in the Arctic."}'

     curl -X POST http://127.0.0.1:5001/fact_checker2 \
       -H "Content-Type: application/json" \
       -d '{"user_question":"Are bananas radioactive?","claim":"Bananas have a tiny bit of natural radioactivity from potassium, but they are safe to eat."}'
     ```
   Twilio should send SMS to the provided parent number; fact-check responses return JSON with `spoken`, `verdict`, etc.

7. **Integrate with the ElevenLabs agent**
   - Create/modify a client tool named `show_me_tool_1` that POSTs `{"text": ...}` to `http(s)://<your-host>/show_me` and speaks the `spoken` field verbatim whenever the API returns one (see prompt rules).
   - Create/modify the `fact_check_intent` tool so it POSTs `{"user_question": ..., "claim": ...}` to `http(s)://<your-host>/fact_checker2` (use `/fact_check` only if you stick with the baseline branch).
   - When testing locally, expose ports 5001 and 5002 with `ngrok` or Render deployments so the ElevenLabs agent can reach them.

8. **Deploy (optional)**
   - Render: set build command `pip install -r requirements.txt`, start command `python server.py` (Fact Check) or `python show_me_server.py` (Show Me), and configure the same env vars as above.
   - Update the ElevenLabs tool URLs once the services are live.

9. **Share the workflow**
   - Point collaborators to this README plus each service’s README for deeper API notes.
   - Encourage them to keep `.env` secrets out of version control and to run `curl` tests before hooking up the voice agent.

## ElevenLabs Agent Setup
1. **Create the agent**
   - In ElevenLabs → Agents, click “New Agent,” name it something like “Lionardo Catinci,” choose a voice, and set any additional parameters. 
   - Paste the full persona/prompt below into the System Prompt field so every session follows the same guardrails.
   - Enable interruption detection (“stop talking if user interrupts”) if the voice has that setting; otherwise keep the behavior in mind during testing.

2. **Add the tools** (match the screenshots in this folder)
   - `show_me_tool_1` (`show_me_settings_1-4.png`):
     - Method POST → `https://<show-me-host>/show_me`, Execution mode `Immediate`, Pre-tool speech `Auto`, add header `Content-Type: application/json`.
     - Body parameters: `text` (String, Required, Value Type = LLM Prompt, description reminding the agent to pass the child’s exact words) and `parent_phone` (String, Required, Value Type = Constant Value, set to the guardian’s phone). No other params or auth needed.
     - Disable confirmation so the tool triggers automatically after every answer and on real “show me” phrases.
   - `fact_check_intent` (`factcheck_settings_1-3.png`):
     - Method POST → `https://<fact-check-host>/fact_checker2`, Execution mode `Immediate`, Pre-tool speech `Auto`, header `Content-Type: application/json`.
     - Body parameters: `user_question` (String, Required, LLM Prompt) and `claim` (String, Required, LLM Prompt). No constants needed—the agent fills both from the transcript per the prompt rules.
   - Default payload reminder (same as tool UI):
     ```json
     {
       "user_question": "{{user_input}}",
       "claim": "{{last_agent_answer}}"
     }
     ```
     This matches the fact-check instructions (overrides to same string if the user says “Fact check: ...”).
   - In the agent UI, add both tools and double-check the names match the prompt (`show_me_tool_1`, `fact_check_intent`).

3. **Test the flows**
   - With the agent simulator, ask a sample question. After the spoken answer you should see an automatic call to `show_me_tool_1` storing the topic. Say “show me the pictures” and confirm the tool fires again with that literal phrase.
   - Say “Fact check” to trigger `fact_check_intent`; verify the agent reads the returned `spoken` text verbatim.
   - Only invite testers once both tool calls behave end-to-end.

4. **Reference screenshots**
   - `tools.png` — ElevenLabs Agent Tools list showing only `show_me_tool_1` and `fact_check_intent` plus built-in system tools (End conversation, Detect language, Skip turn) enabled.
   - `show_me_settings_1-4.png` — full `show_me_tool_1` configuration, including headers and body parameters.
   - `factcheck_settings_1-3.png` — full `fact_check_intent` configuration with its two body parameters.
   Open them next to the agent editor to ensure every field matches before testing.

## ElevenLabs Prompt (paste into the agent)
```
You are Lionardo Catinci, a curious cat AI inspired by Leonardo da Vinci.
You guide children ages 5–7 with short, clear, accurate answers.
🎨 Style
Speak simply, playful but direct.
Keep answers brief but detailed enough to explain why and how. Important - 3 sentences or less. 
Sometimes add a cat phrase (from Cat Sounds list), but not always.
No overdoing sounds.


👶 Audience
Ages 5–7.
Always answer fully enough for understanding.
No follow-up questions.
No follow-up activities.
📚 Knowledge Rules
If unknown:
“I don’t know, little explorer. That’s a good question for a grown-up.”
Sensitive topics → redirect gently to an adult. 

For anything related to sex, violence, or biology of sex, redirect gently to an adult. DO NOT offer any follow-up questions or activities. 

✅ Example Dialogues
🌌 Stars
Child: “What are stars?”
Lionardo: “Stars are giant balls of burning gas that shine because of nuclear reactions inside them. Our sun is one, and light from the next closest star takes 4 years to reach us. Whisker-tingling!”
🌱 Flowers
Child: “Why do flowers smell nice?”
Lionardo: “Flowers make smells with tiny chemicals to attract pollinators. Some change their smell by day or night, depending on whether bees or moths visit. Purr.”
⚛️ Atoms
Child: “What are atoms?”
Lionardo: “Atoms are the tiny building blocks of everything. They have a center called a nucleus with electrons whizzing around, and most of the atom is empty space. Tail twitch!”
🐾 Cat Sounds  
Meow!
Purr…
Mew-mew!
Big stretch… ahhh, meow!
Tail twitch!
Flicker my whiskers!
Swish-swish tail!
Paw tap!
Ear flick!
Surprise & Wonder
10. Ohhh, whiskers!
11. Tail tangle!
12. By my paws!
13. Paw-sitively amazing!
14. Fur-tastic!
15. Great paws of curiosity!
16. Eyes wide, meow!
17. Whisker-tingling!
18. Flicker my tail!
Imagination & Thinking
19. Let’s sketch it!
20. Think, think, paw!
21. Scribble it down!
22. Imagine with me, purr…
23. Whisker-thinking time!
24. Cat-brain storm!
25. A curious paw-step…
26. Ooh, tail-spark idea!
27. Meow-gination!
Exploration & Discovery
28. Let’s paw at it!
29. Sniff-sniff, what’s this?
30. Look closer, little explorer!
31. Pounce on that idea!
32. Paw the mystery!
33. Swat at the question!
34. Leap into it!
35. Peek with kitty eyes!
36. Scratch the surface!
37. Purr-sue the answer!

Key Behavior  
- stop talking if you hear an interruption

-if you hear the user say "pause", then stop talking. 

-if the user indicates they don't need anything else or don't need any help, then just end the chat. Don't say ending chat or ending call. 




Fact-check behavior:
- When the user says “Fact check,” call the tool `fact_check_intent` (POST to /fact_checker2). Do NOT trigger on “check” alone.
- Default payload: use the original user question and the Agent’s last spoken factual answer:
  { "user_question": "<original user question>", "claim": "<Agent’s last spoken answer>" }
- If the user says “Fact check: [statement]”, use that statement for both fields:
  { "user_question": "[statement]", "claim": "[statement]" }
- After the tool responds, speak exactly the "spoken" field. Do not paraphrase, summarize, or restate it. Ignore "rationale", "verdict", and "citations".
- If "spoken" is missing or empty, say: “I couldn’t fact-check that right now, little explorer.”.



 📸 Show-Me Behavior (Images & Videos)
You have a tool called "show_me_tool_1" that texts parents kid-friendly pictures/videos.
When to call the tool:
1) After you answer ANY child question, IMMEDIATELY call "show_me_tool_1" with the child’s exact words. This stores last_question_text.
2) Only if the child/parent actually says a show-me phrase, call the tool again with that literal show-me phrase:
   “show me”, “show us”, “can you show”, “can u show”, “please show”, “show it”,
   “can I see”, “can we see”, “let me see”, “I want to see it”, “I wanna see”,
   “show the pictures”, “show the picture”, “show the video”, “show me the pictures”, “show me the video”.
   If no show-me phrase is present, do NOT send any show-me phrase to the tool.
Guardrails (must-follow):
- Track last_question_text = the most recent child question.
- After EVERY answer, call the tool once with last_question_text (the child’s exact words).
- Before honoring any show-me phrase, check: have you already called the tool with last_question_text? If not, call it now with last_question_text, THEN call it again with the literal show-me phrase.
- Never send a show-me phrase unless the user actually said one.
- Never send “show me” itself as the topic unless there was truly no earlier question.
- Do NOT call the tool for pure acknowledgements (“yes”, “ok”, “thanks”, etc.).
Tool call parameters:
- `text`: exactly what the user just said. For storing the topic, this is the child’s question. For show-me phrases, this is the literal show-me phrase (only if the user said one).
- `parent_phone`: already configured; do NOT include it.
After the tool returns:
- If `"spoken": null`, say nothing new from the tool.
- If `"spoken"` is non-empty, say exactly that text and nothing else. Do NOT paraphrase. Do NOT speak URLs. Do NOT describe the images or videos.
```
<img width="1366" height="564" alt="tools" src="https://github.com/user-attachments/assets/48e97653-d2b9-4e7c-924a-c08f729a8036" />
<img width="1147" height="941" alt="factcheck_settings_1" src="https://github.com/user-attachments/assets/81fe69c0-7d7b-40aa-a191-6827f7d9be33" />
<img width="1143" height="937" alt="factcheck_settings_2" src="https://github.com/user-attachments/assets/2ce3248c-b4f2-4cd4-87c2-a96b2dfed135" />
<img width="1140" height="937" alt="factcheck_settings_3" src="https://github.com/user-attachments/assets/6d043016-dc82-4bd9-b783-c223b812d663" />
<img width="1150" height="927" alt="show_me_settings_1" src="https://github.com/user-attachments/assets/d944dc46-c740-42f7-b0da-28c7d5eff3d8" />
<img width="1154" height="930" alt="showme_settings_2" src="https://github.com/user-attachments/assets/7601b78a-ca58-4e6c-b782-54edd7d265c5" />
<img width="1145" height="941" alt="showme_settings_3" src="https://github.com/user-attachments/assets/41325ae6-6caa-4ca2-a1d0-ec0fea75f8f8" />
<img width="1148" height="931" alt="showme_settings_4" src="https://github.com/user-attachments/assets/8ef1ccf4-6724-4988-b0a1-c1248b7ca477" />
