---
title: "LLM Wiki pod maską: Python, Obsidian i Gemini"
date: 2026-07-08
---

W pierwszym [wpisie na LinkedIn](https://www.linkedin.com/posts/marcinpendolski_dataanalytics-ai-knowledgemanagement-share-7470182601656565760-lkeY/?utm_source=share&utm_medium=member_desktop&rcm=ACoAACLNJl4BEVvx8Dyrv3vQKWalkk_oHr4oJEU) opisałem koncepcję własnego LLM Wiki jako alternatywę dla standardowego podejścia RAG. Zamiast wektoryzować i przeszukiwać dokumenty przy każdym zapytaniu, system jednorazowo kompiluje i strukturyzuje wiedzę.

Poniżej omawiam architekturę i kod z repozytorium llm_wiki na GitHubie. Architektura celowo odrzuca bazy wektorowe na rzecz pełnego okna kontekstowego modeli Google Gemini i ścisłej strukturyzacji danych wejściowych.

## Struktura danych i wymuszanie schematu (pydantic_models.py)

Domyślnym wyjściem modelu LLM jest nieustrukturyzowany tekst, co utrudnia automatyzację. Aby zintegrować model z systemem plików, stosuję bibliotekę Pydantic do zdefiniowania sztywnego schematu wyjściowego (JSON). Model działa tu jako agent decyzyjny operujący na notatkach:

```python
# --- Python ---
from pydantic import BaseModel, Field
from typing import List

class WikiAction(BaseModel):
    action_type: str = Field(description="Typ akcji: 'CREATE', 'UPDATE' lub 'LOG'")
    file_path: str = Field(description="Ścieżka do pliku w folderze wiki/, np. 'wiki/sztuczna_inteligencja.md'")
    title: str = Field(description="Tytuł notatki (używany jako nazwa pliku i frontmatter)")
    content: str = Field(description="Pełna treść notatki w formacie Markdown z linkami [[dwunawiasowymi]]. Jeśli UPDATE, podaj scaloną, zaktualizowaną treść.")
    log_message: str = Field(description="Krótki opis zmiany do zapisania w logu")

class WikiCompilationResult(BaseModel):
    actions: List[WikiAction] = Field(description="Lista operacji na plikach wiki niezbędnych do przetworzenia źródła")
```

## Asymilacja i strukturyzacja wiedzy (kompilacja_bazy.py)
Głównym interfejsem bazy jest Obsidian. Nowa, surowa notatka wrzucona do folderu raw wyzwala proces kompilacji. Zanim model przetworzy nowy plik, funkcja get_current_wiki_context() ładuje zmapowaną strukturę istniejącego repozytorium.

Dzięki temu i narzuconemu schematowi WikiCompilationResult, model podejmuje decyzję kontekstową: tworzy nowy plik lub aktualizuje treść w już istniejącym.

```python
# --- Python ---
import os
import glob
from google import genai
from google.genai import types
from pydantic_models import WikiCompilationResult 
from dotenv import load_dotenv

load_dotenv() # Klucze API ładowane bezpiecznie z pliku .env (wykluczonego w .gitignore)
client = genai.Client()

def get_current_wiki_context() -> str:
    """Zbiera strukturę i spis treści obecnego wiki."""
    context = "OBECNY STAN WIKI:\n"
    wiki_files = glob.glob("/Users/mpendolski/Obsidian/llm-wiki/wiki/*.md")
    
    if not wiki_files:
        return context + "Wiki jest puste. Brak istniejących notatek."
        
    for file_path in wiki_files:
        with open(file_path, "r", encoding="utf-8") as f:
            lines = [f.readline() for _ in range(15)]
            context += f"--- PLIK: {file_path} ---\n" + "".join(lines) + "\n"
    return context

def compile_source_to_wiki(raw_file_path: str):
    print(f"⚡ Przetwarzanie pliku: {raw_file_path}...")
    
    with open(raw_file_path, "r", encoding="utf-8") as f:
        raw_content = f.read()
        
    wiki_context = get_current_wiki_context()
    system_instruction = open("/Users/mpendolski/Obsidian/llm-wiki/AGENTS.md", "r", encoding="utf-8").read()

    user_prompt = f"""
    Zintegruj poniższy materiał źródłowy z moim Wiki.
    
    {wiki_context}
    
    MATERIAŁ ŹRÓDŁOWY DO PRZETWORZENIA:
    {raw_content}
    """

    response = client.models.generate_content(
        model='gemini-2.5-flash',
        contents=user_prompt,
        config=types.GenerateContentConfig(
            system_instruction=system_instruction,
            temperature=0.1,  
            response_mime_type="application/json",
            response_schema=WikiCompilationResult,
        ),
    )

    result: WikiCompilationResult = response.parsed
    # [Dalsza część skryptu zapisuje wygenerowane pliki Markdown i aktualizuje log]
```

## Odpytywanie bazy bez halucynacji (odpytanie_bazy.py)
Zamiast wyszukiwania semantycznego po fragmentach (chunkach), skrypt ładuje pełną zawartość skompilowanych notatek z Obsidiana wprost do okna kontekstowego Gemini. System prompt ściśle ogranicza model do dostarczonych danych i wymusza tagowanie pojęć (np. [[Nazwa Notatki]]). Niska temperatura (0.2) minimalizuje ryzyko halucynacji i zapewnia deterministyczne wyjście.

```python
# --- Python ---
import os
import glob
from google import genai
from google.genai import types
from dotenv import load_dotenv

load_dotenv()
client = genai.Client()

def load_compiled_wiki() -> str:
    """Ładuje PEŁNĄ zawartość wszystkich skompilowanych notatek wiki."""
    full_wiki_content = ""
    wiki_files = glob.glob("/Users/mpendolski/Obsidian/llm-wiki/wiki/*.md")
    
    for file_path in wiki_files:
        if "log.md" in file_path:
            continue
            
        with open(file_path, "r", encoding="utf-8") as f:
            filename = os.path.basename(file_path).replace(".md", "")
            full_wiki_content += f"\n=== NOTATKA: {filename} ===\n"
            full_wiki_content += f.read() + "\n"
            
    return full_wiki_content

def ask_wiki(question: str):
    wiki_data = load_compiled_wiki()
    
    system_instruction = """
    Jesteś ekspertem analizującym bazę wiedzy użytkownika. 
    Twoje odpowiedzi muszą opierać się WYŁĄCZNIE na dostarczonych notatkach z sekcji KOMPILACJA WIKI.
    Jeśli w bazie nie ma informacji potrzebnych do odpowiedzi, zwróć komunikat: 'Brak danych w wiki'.
    Kiedy odwołujesz się do pojęcia, które istnieje w bazie, użyj formatu [[Nazwa Notatki]], tak jak w Obsidianie.
    """
    
    prompt = f"""
    KOMPILACJA WIKI:
    {wiki_data}
    
    PYTANIE UŻYTKOWNIKA:
    {question}
    """
    
    response = client.models.generate_content(
        model='gemini-3.5-flash',
        contents=prompt,
        config=types.GenerateContentConfig(
            system_instruction=system_instruction,
            temperature=0.2, 
        ),
    )
    print(response.text)
```

## Podsumowanie

LLM Wiki automatyzuje asymilację luźnych notatek i zamienia je w ustrukturyzowaną bazę Markdown, pozwalając na analizę danych i wyciąganie wniosków w oparciu o pełny kontekst projektowy.
Cały udostępniony kod znajdziesz w repozytorium: [GitHub - llm_wiki](https://github.com/PendolM/llm_wiki).