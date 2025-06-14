# findinfo
streamlit_code = """
import streamlit as st
import pdfplumber
import docx
from bs4 import BeautifulSoup
import spacy
from keybert import KeyBERT
from transformers import pipeline

nlp = spacy.load("en_core_web_sm")
kw_model = KeyBERT()
summarizer = pipeline("summarization", model="sshleifer/distilbart-cnn-12-6")

def extract_text_from_pdf(file):
    text = ""
    with pdfplumber.open(file) as pdf:
        for page in pdf.pages:
            page_text = page.extract_text()
            if page_text:
                text += page_text + "\\n"
    return text

def extract_text_from_docx(file):
    doc = docx.Document(file)
    return "\\n".join([para.text for para in doc.paragraphs if para.text.strip()])

def extract_text_from_html(file):
    html = file.read().decode("utf-8")
    soup = BeautifulSoup(html, "html.parser")
    return soup.get_text()

def extract_summary(text, max_len=150, min_len=40):
    if not text.strip():
        return "No summary (empty or unreadable content)"
    try:
        summary = summarizer(text[:1024], max_length=max_len, min_length=min_len, do_sample=False)
        return summary[0]['summary_text']
    except Exception as e:
        return f"Error: {str(e)}"

def extract_keywords(text, top_n=10):
    return kw_model.extract_keywords(text, top_n=top_n)

def extract_entities(text):
    doc = nlp(text)
    return [(ent.text, ent.label_) for ent in doc.ents]


st.set_page_config(page_title="Document Info Extractor", layout="wide")
st.title("📄 Document Information Extractor")
st.markdown("Upload a document and extract summary, keywords, and named entities.")

uploaded_file = st.file_uploader("Upload your document (.pdf, .docx, .html, .txt)", type=["pdf", "docx", "html", "txt"])

if uploaded_file:
    file_type = uploaded_file.name.split(".")[-1]

    if file_type == "pdf":
        text = extract_text_from_pdf(uploaded_file)
    elif file_type == "docx":
        text = extract_text_from_docx(uploaded_file)
    elif file_type == "html":
        text = extract_text_from_html(uploaded_file)
    elif file_type == "txt":
        text = uploaded_file.read().decode("utf-8")
    else:
        st.error("Unsupported file format.")
        text = ""

    if text:
        st.subheader("📚 Extracted Text")
        st.text_area("Text", text[:2000], height=300)

        with st.spinner("Processing document..."):
            summary = extract_summary(text)
            keywords = extract_keywords(text)
            entities = extract_entities(text)

        st.subheader("📝 Summary")
        st.write(summary)

        st.subheader("🔑 Keywords")
        st.write([kw[0] for kw in keywords])
        
        max_ents = st.slider("How many named entities to display?", min_value=5, max_value=100, value=20, step=5)


        st.subheader("🏷️ Named Entities")
        if entities:
            st.write(f"Showing top {min(max_ents, len(entities))} entities:")
            for ent, label in entities[:max_ents]:
                st.write(f"- **{ent}** : _{label}_")
        else:
            st.write("No named entities found.")

        
    else:
        st.warning("No text could be extracted from the file.")
"""

with open("app.py", "w") as f:
    f.write(streamlit_code)



   
