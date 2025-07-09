# pubmed-paper-fetch
import requests
import csv
from typing import List, Optional, Dict, Any, Set, Tuple
import xml.etree.ElementTree as ET

# Constants for identifying non-academic/biotech affiliations
PHARMA_KEYWORDS = [
    "pharma", "biotech", "therapeutic", "laboratories", "inc", "corp", "company", "llc", "gmbh", "ltd", "s.a.", "nv", "co.", "plc"
]
NON_ACADEMIC_KEYWORDS = PHARMA_KEYWORDS + [
    "private", "hospital", "clinic", "institute", "center"
]
ACADEMIC_KEYWORDS = [
    "university", "institute of technology", "college", "school of medicine", "faculty", "department", "universität", "universite", "universidade", "universidad"
]


class Paper:
    def __init__(
        self,
        pubmed_id: str,
        title: str,
        publication_date: str,
        non_academic_authors: List[str],
        company_affiliations: List[str],
        corresponding_email: str
    ):
        self.pubmed_id = pubmed_id
        self.title = title
        self.publication_date = publication_date
        self.non_academic_authors = non_academic_authors
        self.company_affiliations = company_affiliations
        self.corresponding_email = corresponding_email

    def to_csv_row(self) -> List[str]:
        return [
            self.pubmed_id,
            self.title,
            self.publication_date,
            "; ".join(self.non_academic_authors),
            "; ".join(self.company_affiliations),
            self.corresponding_email
        ]


class PaperFetcher:
    BASE_ESEARCH = "https://eutils.ncbi.nlm.nih.gov/entrez/eutils/esearch.fcgi"
    BASE_EFETCH = "https://eutils.ncbi.nlm.nih.gov/entrez/eutils/efetch.fcgi"

    def __init__(self, email: Optional[str] = None, debug: bool = False):
        self.email = email
        self.debug = debug

    def fetch_pubmed_ids(self, query: str, max_results: int = 100) -> List[str]:
        params = {
            "db": "pubmed",
            "term": query,
            "retmax": max_results,
            "retmode": "json"
        }
        if self.email:
            params["email"] = self.email
        resp = requests.get(self.BASE_ESEARCH, params=params)
        if self.debug:
            print(f"ESearch URL: {resp.url}")
        resp.raise_for_status()
        ids = resp.json()["esearchresult"]["idlist"]
        return ids

    def fetch_papers(self, pmids: List[str]) -> List[Paper]:
        if not pmids:
            return []
        params = {
            "db": "pubmed",
            "id": ",".join(pmids),
            "retmode": "xml"
        }
        if self.email:
            params["email"] = self.email
        resp = requests.get(self.BASE_EFETCH, params=params)
        if self.debug:
            print(f"EFetch URL: {resp.url}")
        resp.raise_for_status()
        root = ET.fromstring(resp.content)
        papers: List[Paper] = []
        for article in root.findall(".//PubmedArticle"):
            paper = self.parse_article(article)
            if paper:
                papers.append(paper)
        return papers

    def parse_article(self, article) -> Optional[Paper]:
        medline = article.find("MedlineCitation")
        article_info = medline.find("Article") if medline is not None else None
        if article_info is None:
            return None
        # Pubmed ID
        pmid = medline.findtext("PMID")
        # Title
        title = article_info.findtext("ArticleTitle", "")
        # Publication Date
        pub_date = ""
        pub_date_elem = article_info.find(".//PubDate")
        if pub_date_elem is not None:
            year = pub_date_elem.findtext("Year", "")
            month = pub_date_elem.findtext("Month", "")
            day = pub_date_elem.findtext("Day", "")
            pub_date = "-".join(filter(None, [year, month, day]))
        # Authors
        author_list_elem = article_info.find("AuthorList")
        non_academic_authors: List[str] = []
        company_affiliations: Set[str] = set()
        corresponding_email = ""
        if author_list_elem is not None:
            for author in author_list_elem.findall("Author"):
                last = author.findtext("LastName", "")
                fore = author.findtext("ForeName", "")
                name = f"{fore} {last}".strip()
                affiliations = [aff.text for aff in author.findall(".//AffiliationInfo/Affiliation") if aff.text]
                for aff in affiliations:
                    if any(kw in aff.lower() for kw in PHARMA_KEYWORDS):
                        company_affiliations.add(aff)
                        non_academic_authors.append(name)
                    elif not any(kw in aff.lower() for kw in ACADEMIC_KEYWORDS):
                        non_academic_authors.append(name)
                    email = self.extract_email(aff)
                    if email and not corresponding_email:
                        corresponding_email = email
        if not company_affiliations:
            return None  # Only include papers with at least one biotech/pharma author
        return Paper(
            pubmed_id=pmid,
            title=title,
            publication_date=pub_date,
            non_academic_authors=sorted(set(non_academic_authors)),
            company_affiliations=sorted(company_affiliations),
            corresponding_email=corresponding_email
        )

    @staticmethod
    def extract_email(text: str) -> str:
        import re
        m = re.search(r"[\w\.-]+@[\w\.-]+", text)
        return m.group(0) if m else ""


def fetch_papers_with_biotech_author(
    query: str, max_results: int = 100, email: Optional[str] = None, debug: bool = False
) -> List[Paper]:
    fetcher = PaperFetcher(email=email, debug=debug)
    pmids = fetcher.fetch_pubmed_ids(query, max_results)
    return fetcher.fetch_papers(pmids)
