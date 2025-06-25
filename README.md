streamlit as st
from Bio import Entrez, Medline
import arxiv
import datetime
import re
import os
import json

# CSS Styling same as before
st.markdown(
    """
    <style>
    .stApp {
        background-color: #aeeafe;
        color: #4285f4;
    }
    h1, h2, h3, h4, h5, h6 {
        color: #4285f4;
    }
    a {
        color: #4285f4;
    }
    </style>
    """,
    unsafe_allow_html=True
)

Entrez.email = "gilgurari@gmail.com"

st.markdown("<h1 style='color: #4285f4;'>Recent Articles in AI and Echocardiography</h1>", unsafe_allow_html=True)

query_input = st.text_input(
    "Search query (used in PubMed and arXiv):",
    '(echocardiography OR "cardiac ultrasound") AND (deep learning OR ai)'
)

search_filter = st.text_input(
    "Optional keyword filter (in title or abstract):"
).lower()

sort_option = st.selectbox(
    "Sort articles by:",
    ["Date (Newest First)", "Date (Oldest First)", "Title (A→Z)", "Title (Z→A)"]
)

DATA_DIR = "data"
os.makedirs(DATA_DIR, exist_ok=True)

def sort_articles(articles, key_date_fn, key_title_fn):
    if sort_option == "Date (Newest First)":
        return sorted(articles, key=key_date_fn, reverse=True)
    elif sort_option == "Date (Oldest First)":
        return sorted(articles, key=key_date_fn)
    elif sort_option == "Title (A→Z)":
        return sorted(articles, key=key_title_fn)
    else:
        return sorted(articles, key=key_title_fn, reverse=True)

def search_pubmed(query, max_results=20):
    handle = Entrez.esearch(db="pubmed", term=query, retmax=max_results)
    record = Entrez.read(handle)
    return record["IdList"]

def fetch_pubmed_details(id_list):
    handle = Entrez.efetch(db="pubmed", id=",".join(id_list), rettype="medline", retmode="text")
    return list(Medline.parse(handle))

def pubmed_to_dict(records):
    # Serialize only needed fields
    out = []
    for r in records:
        out.append({
            'TI': r.get('TI'),
            'AU': r.get('AU'),
            'AB': r.get('AB'),
            'DP': r.get('DP'),
            'PMID': r.get('PMID'),
            'AD': r.get('AD'),
        })
    return out

@st.cache_data(ttl=604800)
def get_pubmed_articles(query):
    pmids = search_pubmed(query, max_results=20)
    if not pmids:
        return []
    records = fetch_pubmed_details(pmids)
    return pubmed_to_dict(records)

def arxiv_to_dict(results):
    out = []
    for r in results:
        out.append({
            'title': r.title,
            'authors': [str(a) for a in r.authors],
            'summary': r.summary,
            'published': r.published.strftime("%Y-%m-%d"),
            'entry_id': r.entry_id,
        })
    return out

def search_arxiv(query, max_results=20):
    search = arxiv.Search(query=query, max_results=max_results, sort_by=arxiv.SortCriterion.Relevance)
    return list(search.results())

@st.cache_data(ttl=604800)
def get_arxiv_articles(query):
    results = search_arxiv(query, max_results=20)
    return arxiv_to_dict(results)

def extract_emails(text):
    if not text:
        return []
    return re.findall(r'\b[\w\.-]+?@\w+?\.\w+?\b', text)

def save_snapshot(pubmed, arxiv, week_str):
    file_path = os.path.join(DATA_DIR, f"articles_{week_str}.json")
    with open(file_path, 'w', encoding='utf-8') as f:
        json.dump({'pubmed': pubmed, 'arxiv': arxiv}, f, ensure_ascii=False, indent=2)

def load_snapshot(week_str):
    file_path = os.path.join(DATA_DIR, f"articles_{week_str}.json")
    if not os.path.exists(file_path):
        return None
    with open(file_path, 'r', encoding='utf-8') as f:
        return json.load(f)

# Calculate current week Monday date string
def get_monday_of_week(date):
    return (date - datetime.timedelta(days=date.weekday())).strftime("%Y-%m-%d")

# Main tabs
tab1, tab2, tab3 = st.tabs(["Current Week", "Past Weeks", "About"])

# Current Week Tab: Fetch and display live data
with tab1:
    st.header("Current Week Articles")

    if query_input.strip():
        pubmed_results = get_pubmed_articles(query_input)
        arxiv_results = get_arxiv_articles(query_input)

        # Filter if needed
        if search_filter:
            pubmed_results = [
                r for r in pubmed_results
                if (search_filter in (r.get("TI") or "").lower()) or (search_filter in (r.get("AB") or "").lower())
            ]
            arxiv_results = [
                r for r in arxiv_results
                if (search_filter in r['title'].lower()) or (search_filter in r['summary'].lower())
            ]

        # Sort helper functions
        def pub_date(r):
            try:
                return int((r.get("DP") or "")[:4])
            except:
                return 0
        def pub_title(r):
            return (r.get("TI") or "").lower()
        def arxiv_date(r):
            try:
                return datetime.datetime.strptime(r['published'], "%Y-%m-%d").timestamp()
            except:
                return 0
        def arxiv_title(r):
            return r['title'].lower()

        pubmed_results = sort_articles(pubmed_results, pub_date, pub_title)
        arxiv_results = sort_articles(arxiv_results, arxiv_date, arxiv_title)

        # Save snapshot for this week
        week_str = get_monday_of_week(datetime.datetime.now())
        save_snapshot(pubmed_results, arxiv_results, week_str)

        st.caption(f"Last refreshed: {datetime.datetime.now().strftime('%Y-%m-%d %H:%M')}")

        tab1a, tab1b = st.tabs(["PubMed", "arXiv"])

        with tab1a:
            st.write(f"PubMed articles found: {len(pubmed_results)}")
            if pubmed_results:
                for record in pubmed_results:
                    with st.expander(record.get("TI", "No title")):
                        authors = record.get("AU", [])
                        st.write("**Authors:**", ", ".join(authors))
                        emails = []
                        if 'AD' in record:
                            emails = extract_emails(" ".join(record.get('AD')))
                        if emails:
                            st.write("**Author emails:**", ", ".join(emails))
                        else:
                            st.write("**Author emails:** Not available")
                        st.write("**Abstract:**", record.get("AB", "No abstract"))
                        st.write("**Date:**", record.get("DP", "No date"))
                        st.write(f"[PubMed Link](https://pubmed.ncbi.nlm.nih.gov/{record.get('PMID')})")
            else:
                st.write("No PubMed articles found.")

        with tab1b:
            st.write(f"arXiv articles found: {len(arxiv_results)}")
            if arxiv_results:
                for result in arxiv_results:
                    with st.expander(result['title']):
                        authors_list = result['authors']
                        st.write("**Authors:**", ", ".join(authors_list))
                        st.write("**Abstract:**", result['summary'])
                        st.write("**Date:**", result['published'])
                        st.write(f"[arXiv Link]({result['entry_id']})")
            else:
                st.write("No arXiv articles found.")
    else:
        st.info("Please enter a search query above.")

# Past Weeks Tab: Select a saved snapshot and display it
with tab2:
    st.header("Browse Past Weeks")

    saved_files = sorted([
        f for f in os.listdir(DATA_DIR) if f.startswith("articles_") and f.endswith(".json")
    ], reverse=True)

    if not saved_files:
        st.write("No past data snapshots found.")
    else:
        # Extract week dates from filenames
        weeks = [f[len("articles_"):-len(".json")] for f in saved_files]
        selected_week = st.selectbox("Select week to view:", weeks)

        snapshot = load_snapshot(selected_week)
        if snapshot:
            pubmed_results = snapshot.get('pubmed', [])
            arxiv_results = snapshot.get('arxiv', [])

            tab2a, tab2b = st.tabs(["PubMed", "arXiv"])

            with tab2a:
                st.write(f"PubMed articles found: {len(pubmed_results)}")
                if pubmed_results:
                    for record in pubmed_results:
                        with st.expander(record.get("TI", "No title")):
                            authors = record.get("AU", [])
                            st.write("**Authors:**", ", ".join(authors))
                            emails = []
                            if 'AD' in record:
                                emails = extract_emails(" ".join(record.get('AD')))
                            if emails:
                                st.write("**Author emails:**", ", ".join(emails))
                            else:
                                st.write("**Author emails:** Not available")
                            st.write("**Abstract:**", record.get("AB", "No abstract"))
                            st.write("**Date:**", record.get("DP", "No date"))
                            st.write(f"[PubMed Link](https://pubmed.ncbi.nlm.nih.gov/{record.get('PMID')})")
                else:
                    st.write("No PubMed articles found.")

            with tab2b:
                st.write(f"arXiv articles found: {len(arxiv_results)}")
                if arxiv_results:
                    for result in arxiv_results:
                        with st.expander(result['title']):
                            authors_list = result['authors']
                            st.write("**Authors:**", ", ".join(authors_list))
                            st.write("**Abstract:**", result['summary'])
                            st.write("**Date:**", result['published'])
                            st.write(f"[arXiv Link]({result['entry_id']})")
                else:
                    st.write("No arXiv articles found.")
        else:
            st.error("Failed to load selected snapshot.")

# About Tab (optional)
with tab3:
    st.header("About")
    st.write("""
        This dashboard searches PubMed and arXiv for articles on AI in echocardiography.
        It saves weekly snapshots locally, allowing you to browse past weeks' articles.
        
        To keep the data updated automatically every week, consider scheduling this app
        on a server or using a scheduler (cron, Windows Task Scheduler) to run it periodically.
    """)

