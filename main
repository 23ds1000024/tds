import os
import sys
import traceback
from io import StringIO
from typing import List

import requests
from fastapi import FastAPI
from fastapi.middleware.cors import CORSMiddleware
from pydantic import BaseModel


app = FastAPI()

app.add_middleware(
    CORSMiddleware,
    allow_origins=["*"],
    allow_credentials=True,
    allow_methods=["*"],
    allow_headers=["*"],
)


class CodeRequest(BaseModel):
    code: str


class CodeResponse(BaseModel):
    error: List[int]
    result: str


def execute_python_code(code: str) -> dict:
    old_stdout = sys.stdout
    sys.stdout = StringIO()

    try:
        exec(code, {})
        output = sys.stdout.getvalue()
        return {"success": True, "output": output}
    except Exception:
        output = traceback.format_exc()
        return {"success": False, "output": output}
    finally:
        sys.stdout = old_stdout


def analyze_error_with_ai(code: str, tb: str) -> List[int]:
    token = "eyJhbGciOiJIUzI1NiJ9.eyJlbWFpbCI6IjIzZHMxMDAwMDI0QGRzLnN0dWR5LmlpdG0uYWMuaW4ifQ.367G4wqhozIj51YTIYCJj0dkrzWUZ0t5pdbkgQR32Po"

    prompt = f"""
Analyze this Python code and traceback.
Return only JSON in this format:
{{"error_lines":[line_numbers]}}

Identify the original user-code line number where the error occurred.

CODE:
{code}

TRACEBACK:
{tb}
"""

    response = requests.post(
        "https://aipipe.org/openrouter/v1/chat/completions",
        headers={
            "Content-Type": "application/json",
            "Authorization": f"Bearer {token}",
        },
        json={
            "model": "google/gemini-2.0-flash-lite-001",
            "messages": [
                {
                    "role": "system",
                    "content": "You identify Python error line numbers. Return only valid JSON.",
                },
                {"role": "user", "content": prompt},
            ],
            "response_format": {"type": "json_object"},
        },
        timeout=30,
    )

    response.raise_for_status()
    data = response.json()
    content = data["choices"][0]["message"]["content"]

    parsed = ErrorAnalysis.model_validate_json(content)
    return parsed.error_lines


class ErrorAnalysis(BaseModel):
    error_lines: List[int]


@app.post("/code-interpreter", response_model=CodeResponse)
def code_interpreter(req: CodeRequest):
    execution = execute_python_code(req.code)

    if execution["success"]:
        return {
            "error": [],
            "result": execution["output"],
        }

    error_lines = analyze_error_with_ai(req.code, execution["output"])

    return {
        "error": error_lines,
        "result": execution["output"],
    }