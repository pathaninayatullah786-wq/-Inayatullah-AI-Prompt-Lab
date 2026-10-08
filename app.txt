
import os
import json
import re

import streamlit as st
from groq import Groq

# ---------------- PAGE SETTINGS ----------------

st.set_page_config(
    page_title="Inayatullah AI Prompt Lab",
    page_icon="🤖",
    layout="wide"
)

st.title("🤖 Inayatullah AI Prompt Lab")
st.write(
    "Explore four AI tools for learning, coding, "
    "question generation, and message classification."
)

st.divider()


# ---------------- API CONNECTION ----------------

def get_api_key():
    """Read the Groq API key from Streamlit secrets or environment."""

    try:
        key = st.secrets.get("GROQ_API_KEY", "")
    except Exception:
        key = ""

    if not key:
        key = os.getenv("GROQ_API_KEY", "")

    return key


def ask_ai(system_prompt, user_prompt):
    """Send a question to the Groq AI model."""

    api_key = get_api_key()

    if not api_key:
        st.error(
            "Groq API key is missing. "
            "Follow Step 7 in the instructions below."
        )
        return None

    try:
        client = Groq(api_key=api_key)

        model = os.getenv(
            "GROQ_MODEL",
            "llama-3.3-70b-versatile"
        )

        response = client.chat.completions.create(
            model=model,
            messages=[
                {
                    "role": "system",
                    "content": system_prompt
                },
                {
                    "role": "user",
                    "content": user_prompt
                }
            ],
            temperature=0.3
        )

        return response.choices[0].message.content

    except Exception as error:
        st.error(
            "The AI request failed. Check your internet, "
            "API key, model availability, and Groq account."
        )
        st.caption(str(error))
        return None


# ---------------- FOUR PROJECT TABS ----------------

advisor_tab, reviewer_tab, mcq_tab, classifier_tab = st.tabs(
    [
        "🎓 Academic Advisor",
        "🐍 Code Reviewer",
        "📝 MCQ Generator",
        "🏷️ Request Classifier"
    ]
)


# ---------------- PROJECT 1 ----------------

with advisor_tab:
    st.subheader("🎓 College Academic Advisor")

    st.write(
        "Ask the AI to explain college subjects "
        "in simple, beginner-friendly language."
    )

    subject = st.text_input(
        "Subject or topic",
        placeholder="Example: Python functions",
        key="advisor_subject"
    )

    question = st.text_area(
        "Your question",
        placeholder="What do you want to understand?",
        key="advisor_question"
    )

    if st.button("Explain My Topic", key="advisor_button"):
        if not subject.strip() or not question.strip():
            st.warning("Please enter both a subject and a question.")
        else:
            with st.spinner("Preparing your explanation..."):
                answer = ask_ai(
                    system_prompt=(
                        "You are a friendly college academic advisor. "
                        "Explain concepts in simple language. "
                        "Use short steps and examples. "
                        "If the question is unclear, ask for clarification."
                    ),
                    user_prompt=(
                        f"Subject: {subject}\n"
                        f"Question: {question}"
                    )
                )

            if answer:
                st.markdown("### AI Explanation")
                st.markdown(answer)


# ---------------- PROJECT 2 ----------------

with reviewer_tab:
    st.subheader("🐍 Strict Python Code Reviewer")

    st.write(
        "Paste your Python code. The AI will review it, "
        "explain errors, and suggest corrected code."
    )

    python_code = st.text_area(
        "Paste your Python code",
        height=240,
        placeholder="print('Hello, world!')",
        key="reviewer_code"
    )

    review_type = st.selectbox(
        "Choose review type",
        [
            "Find errors and correct the code",
            "Explain the code line by line",
            "Improve readability",
            "Check logic and edge cases"
        ]
    )

    if st.button("Review My Code", key="reviewer_button"):
        if not python_code.strip():
            st.warning("Please paste some Python code first.")
        else:
            with st.spinner("Reviewing your code..."):
                answer = ask_ai(
                    system_prompt=(
                        "You are a careful Python code reviewer. "
                        "Identify syntax errors, runtime errors, "
                        "and logic problems when possible. "
                        "Explain each issue simply. "
                        "When asked to correct code, provide a complete "
                        "corrected version in a Python code block. "
                        "Never claim code was executed or tested unless "
                        "it actually was."
                    ),
                    user_prompt=(
                        f"Review request: {review_type}\n\n"
                        f"Python code:\n{python_code}"
                    )
                )

            if answer:
                st.markdown("### Review Result")
                st.markdown(answer)


# ---------------- PROJECT 3 ----------------

with mcq_tab:
    st.subheader("📝 MCQ Generator")

    st.write(
        "Generate multiple-choice questions with answers "
        "and explanations in JSON format."
    )

    mcq_topic = st.text_input(
        "Topic",
        placeholder="Example: Computer Networks",
        key="mcq_topic"
    )

    number_of_questions = st.number_input(
        "Number of questions",
        min_value=1,
        max_value=10,
        value=5,
        step=1
    )

    difficulty = st.selectbox(
        "Difficulty",
        ["Easy", "Medium", "Hard"],
        key="mcq_difficulty"
    )

    if st.button("Generate MCQs", key="mcq_button"):
        if not mcq_topic.strip():
            st.warning("Please enter a topic.")
        else:
            with st.spinner("Generating your questions..."):
                answer = ask_ai(
                    system_prompt=(
                        "You are an educational question generator. "
                        "Return ONLY valid JSON. Do not use Markdown "
                        "or code fences. Use this exact structure: "
                        '{"topic":"topic","questions":['
                        '{"question":"...","options":{"A":"...",'
                        '"B":"...","C":"...","D":"..."},'
                        '"correct_answer":"A",'
                        '"explanation":"..."}]}. '
                        "The correct_answer must be A, B, C, or D. "
                        "Include exactly four options per question."
                    ),
                    user_prompt=(
                        f"Topic: {mcq_topic}\n"
                        f"Difficulty: {difficulty}\n"
                        f"Generate exactly {number_of_questions} "
                        "multiple-choice questions."
                    )
                )

            if answer:
                # Remove code fences if the AI accidentally adds them.
                cleaned = re.sub(
                    r"^\s*```(?:json)?\s*|\s*```\s*$",
                    "",
                    answer,
                    flags=re.IGNORECASE
                ).strip()

                try:
                    data = json.loads(cleaned)
                    questions = data.get("questions", [])

                    valid = (
                        isinstance(questions, list)
                        and len(questions) == number_of_questions
                        and all(
                            isinstance(item, dict)
                            and isinstance(item.get("options"), dict)
                            and set(item["options"].keys())
                                == {"A", "B", "C", "D"}
                            and item.get("correct_answer")
                                in {"A", "B", "C", "D"}
                            for item in questions
                        )
                    )

                    if valid:
                        st.success("Valid JSON generated!")
                        st.json(data)
                    else:
                        st.warning(
                            "The response is JSON, but some questions "
                            "do not match the required structure."
                        )
                        st.code(cleaned, language="json")

                except json.JSONDecodeError:
                    st.error(
                        "The AI did not return valid JSON this time. "
                        "Please try again."
                    )
                    st.code(answer)


# ---------------- PROJECT 4 ----------------

with classifier_tab:
    st.subheader("🏷️ Few-Shot Request Classifier")

    st.write(
        "The AI classifies a message into one of five categories."
    )

    st.caption(
        "Examples: 'Fix my Python code' → Coding; "
        "'Explain photosynthesis' → Study."
    )

    message = st.text_area(
        "Enter a message to classify",
        placeholder="Example: Help me prepare for a job interview",
        key="classifier_message"
    )

    if st.button("Classify Message", key="classifier_button"):
        if not message.strip():
            st.warning("Please enter a message.")
        else:
            with st.spinner("Classifying your message..."):
                answer = ask_ai(
                    system_prompt=(
                        "Classify the user's message into exactly one "
                        "category: Coding, Study, Career, Writing, "
                        "or General. Use these examples: "
                        "'Fix my Python bug' = Coding; "
                        "'Explain a science topic' = Study; "
                        "'Prepare for an interview' = Career; "
                        "'Write a formal email' = Writing; "
                        "'Tell me a joke' = General. "
                        "Return ONLY valid JSON in this format: "
                        '{"category":"Coding","reason":"short reason"}. '
                        "Use only the five allowed category names."
                    ),
                    user_prompt=message
                )

            if answer:
                cleaned = re.sub(
                    r"^\s*```(?:json)?\s*|\s*```\s*$",
                    "",
                    answer,
                    flags=re.IGNORECASE
                ).strip()

                try:
                    result = json.loads(cleaned)
                    allowed = {
                        "Coding", "Study", "Career", "Writing", "General"
                    }

                    if (
                        isinstance(result, dict)
                        and result.get("category") in allowed
                        and isinstance(result.get("reason"), str)
                    ):
                        st.success(
                            f"Category: {result['category']}"
                        )
                        st.write(
                            f"Reason: {result['reason']}"
                        )
                        st.json(result)
                    else:
                        st.warning(
                            "The response did not match the required "
                            "classification format."
                        )
                        st.code(cleaned, language="json")

                except json.JSONDecodeError:
                    st.error("The AI did not return valid JSON.")
                    st.code(answer)


# ---------------- FOOTER ----------------

st.divider()
st.caption(
    "Inayatullah AI Prompt Lab | Built with Python, "
    "Streamlit, and Groq"
)