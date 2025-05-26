# Screenshot_Generated_Test_Cases
import google.generativeai as genai
import pandas as pd
import json
import re
import getpass
from google.colab import files
import base64
from PIL import Image
from io import BytesIO

# 🔐 Get Gemini API key
api_key = getpass.getpass("🔐 Enter your Gemini API key: ")
genai.configure(api_key=api_key)
model = genai.GenerativeModel('gemini-2.0-flash')

# 📂 Upload image files (screenshots)
print("📂 Upload application screenshots (in order):")
uploaded = files.upload()
image_files = sorted(uploaded.keys())  # ensure processing in order

# Function: Encode image to base64
def encode_image(file_bytes):
    return base64.b64encode(file_bytes).decode("utf-8")

# Function: Build the prompt
def build_prompt(index, image_names):
    prompt = f"""
You are a QA engineer. Based on screenshot {index+1} of the application (out of {len(image_names)}), generate **domain-specific functional test cases**.

Your task:
- Understand this screen’s purpose.
- Infer app flow from order of images: {', '.join(image_names)}.
- Identify **application domain** and tailor test cases accordingly.
- Return output in **strict JSON only** (no markdown, notes, or extra text).

Format:
{{
  "test_cases": [
    {{
      "test_scenario": "Scenario description",
      "test_case": "Test case description",
      "steps": ["Step 1", "Step 2", "Step 3"],
      "expected_result": "Expected outcome"
    }}
  ]
}}

This is screen {index+1}: {image_names[index]}
"""
    return prompt.strip()

# Function: Call Gemini with image and prompt
def generate_test_cases_from_image(image_bytes, prompt):
    try:
        response = model.generate_content([
            {"text": prompt},
            {
                "inline_data": {
                    "mime_type": "image/png",
                    "data": encode_image(image_bytes)
                }
            }
        ])
        response_text = response.text.strip()
        response_text = re.sub(r'^```json|```$', '', response_text, flags=re.IGNORECASE).strip()
        data = json.loads(response_text)
        return data.get("test_cases", [])
    except json.JSONDecodeError as e:
        print(f"❌ JSON error: {e}\n🔎 Raw:\n{response_text}")
        return []
    except Exception as e:
        print(f"❌ Gemini error: {e}")
        return []

# Function: Flatten test cases to DataFrame
def test_cases_to_dataframe(test_cases, screen_name):
    rows = []
    for tc in test_cases:
        steps = tc.get("steps", [])
        if isinstance(steps[0], dict):
            steps = [s.get("description", "") for s in steps]
        rows.append({
            "Screen Name": screen_name,
            "Test Scenario": tc.get("test_scenario", ""),
            "Test Case": tc.get("test_case", ""),
            "Steps": "\n".join(steps),
            "Expected Result": tc.get("expected_result", "")
        })
    return pd.DataFrame(rows)

# 📦 Main logic
all_test_cases = pd.DataFrame()

for i, file_name in enumerate(image_files):
    print(f"🔍 Processing {file_name} ({i+1}/{len(image_files)})...")
    image_bytes = uploaded[file_name]
    prompt = build_prompt(i, image_files)
    test_cases = generate_test_cases_from_image(image_bytes, prompt)
    df = test_cases_to_dataframe(test_cases, screen_name=file_name)
    all_test_cases = pd.concat([all_test_cases, df], ignore_index=True)

# 📝 Save to Excel
output_file = "Screenshot_Generated_Test_Cases.xlsx"
all_test_cases.to_excel(output_file, index=False)
print(f"✅ Saved test cases to {output_file}")
files.download(output_file)
