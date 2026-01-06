# labelsense-ai


# food_copilot_gemini.py
# Single-file AI reasoning engine using Google Gemini

import google.generativeai as genai
from typing import List, Dict

# ---------------------------------
# GEMINI CONFIG
# ---------------------------------

genai.configure(api_key="AIzaSyB7dWP5vMiUPt3eP4AoRfb3f-u7JL_XvlQ")

MODEL = genai.GenerativeModel(
    model_name="gemini-1.5-pro",
    system_instruction="""
You are an AI food ingredient copilot.

Rules:
- Do NOT label food as good or bad
- Use calm, human metaphors (glass, spoon, objects)
- Explain trade-offs, not judgments
- Admit uncertainty honestly
- Be concise unless deeper detail is requested
"""
)

# ---------------------------------
# INGREDIENT KNOWLEDGE BASE
# ---------------------------------

ALLERGENS = {
    "peanut": "peanut",
    "groundnut": "peanut",
    "milk": "dairy",
    "soy": "soy",
    "gluten": "gluten"
}

FUNCTIONAL = ["emulsifier", "stabilizer", "preservative"]
COSMETIC = ["color", "colour", "flavour", "flavor"]

# ---------------------------------
# USER MEMORY (FEEDBACK LEARNING)
# ---------------------------------

class UserMemory:
    def __init__(self):
        self.flags = {}      # ingredient -> avoid
        self.feedback = []

    def add_feedback(self, ingredient: str, sentiment: str):
        self.feedback.append({
            "ingredient": ingredient,
            "sentiment": sentiment
        })

        if sentiment in ["disliked", "issue"]:
            self.flags[ingredient.lower()] = "avoid"

    def should_prioritize(self, ingredient: str) -> bool:
        return self.flags.get(ingredient.lower()) == "avoid"

# ---------------------------------
# INGREDIENT CLASSIFICATION
# ---------------------------------

def classify_ingredient(name: str) -> Dict:
    n = name.lower()

    for key in ALLERGENS:
        if key in n:
            return {
                "category": "context_sensitive",
                "purpose": "common_allergen",
                "label": ALLERGENS[key],
                "icon": "⚠️"
            }

    for key in FUNCTIONAL:
        if key in n:
            return {
                "category": "functional",
                "purpose": "product_stability",
                "icon": "⚙️"
            }

    for key in COSMETIC:
        if key in n:
            return {
                "category": "cosmetic",
                "purpose": "appearance_only",
                "icon": "🎨"
            }

    return {
        "category": "nourishing",
        "purpose": "adds_nutrition",
        "icon": "🥦"
    }

# ---------------------------------
# PROMPT BUILDER
# ---------------------------------

def build_prompt(
    ingredients: List[str],
    priority: str | None,
    quick: bool
) -> str:

    prompt = f"Ingredients: {', '.join(ingredients)}.\n"

    if priority:
        prompt += (
            f"The user previously had a negative experience with {priority}. "
            "Highlight this first.\n"
        )

    if quick:
        prompt += (
            "Give a 10-second explanation. "
            "Focus on the single most important insight."
        )
    else:
        prompt += (
            "Give a deeper explanation including trade-offs "
            "and areas of uncertainty."
        )

    return prompt

# ---------------------------------
# MAIN COPILOT ENGINE
# ---------------------------------

class FoodCopilot:
    def __init__(self):
        self.memory = UserMemory()

    def analyze(
        self,
        ingredients: List[str],
        quick: bool = True
    ) -> Dict:

        ingredient_meta = []
        questions = []

        for ing in ingredients:
            meta = classify_ingredient(ing)
            ingredient_meta.append({
                "name": ing,
                **meta
            })

            if meta["category"] == "context_sensitive":
                questions.append({
                    "type": "health_check",
                    "question": (
                        f"This product contains {meta['label']}. "
                        "Should I flag this for you?"
                    )
                })

        priority = None
        for ing in ingredients:
            if self.memory.should_prioritize(ing):
                priority = ing
                break

        prompt = build_prompt(
            ingredients=ingredients,
            priority=priority,
            quick=quick
        )

        response = MODEL.generate_content(prompt)

        return {
            "summary": response.text,
            "ingredients": ingredient_meta,
            "questions": questions,
            "visual_hints": self._visual_hints(ingredients)
        }

    def record_feedback(self, ingredient: str, sentiment: str):
        self.memory.add_feedback(ingredient, sentiment)

    # ---------------------------------
    # VISUAL SIGNALS FOR FRONTEND
    # ---------------------------------

    def _visual_hints(self, ingredients: List[str]) -> List[Dict]:
        visuals = []

        for ing in ingredients:
            if "sugar" in ing.lower():
                visuals.append({
                    "type": "quantity_metaphor",
                    "ingredient": "sugar",
                    "visual": "half_glass_of_sugar"
                })

        return visuals

# ---------------------------------
# FRONTEND JSON CONTRACT (REFERENCE)
# ---------------------------------

"""
Returned JSON structure:

{
  "summary": "The main thing here is sugar...",
  "ingredients": [
    {
      "name": "Emulsifier E471",
      "category": "functional",
      "purpose": "product_stability",
      "icon": "⚙️"
    }
  ],
  "questions": [
    {
      "type": "health_check",
      "question": "This product contains peanuts. Should I flag this for you?"
    }
  ],
  "visual_hints": [
    {
      "type": "quantity_metaphor",
      "ingredient": "sugar",
      "visual": "half_glass_of_sugar"
    }
  ]
}
"""
