import streamlit as st
import requests, json, re, time, io, csv
from urllib.parse import quote

# -----------------------------
# App settings
# -----------------------------
st.set_page_config(page_title="Torah Matrix (ELS)", page_icon="🔍", layout="wide")
BOOKS = ["Genesis", "Exodus", "Leviticus", "Numbers", "Deuteronomy"]
INDEX_API = "https://www.sefaria.org/api/index/{book}"
TEXTS_API = "https://www.sefaria.org/api/texts/{ref}?lang=he&with_text=0&pad=0"

# -----------------------------
# Hebrew normalization
# -----------------------------
FINALS = {"\u05DA": "\u05DB", "\u05DD": "\u05DE", "\u05DF": "\u05E0", "\u05E3": "\u05E4", "\u05E5": "\u05E6"}

def normalize_hebrew(raw: str) -> str:
    # Keep only consonants א-ת; strip vowels, cantillation, punctuation, maqaf
    out = []
    for ch in raw:
        code = ord(ch)
        if 0x05D0 <= code <= 0x05EA:  # Hebrew letters only
            out.append(FINALS.get(ch, ch))
        # else: skip (niqqud, ta'amim, spaces, punctuation)
    return "".join(out)

# -----------------------------
# English -> Hebrew transliteration (simple + practical)
# -----------------------------
DIGRAPHS = {
    "TS": "צ", "TZ": "צ",
    "SH": "ש", "CH": "ח", "KH": "כ", "PH": "פ", "TH": "ת"
}
SINGLE = {
    # Vowels -> yod/vav as carriers (simple heuristic)
    "A": "י", "E": "י", "I": "י", "Y": "י",
    "O": "ו", "U": "ו", "W": "ו",
    # Consonants
    "B": "ב", "V": "ב", "G": "ג", "D": "ד", "H": "ה", "Z": "ז",
    "C": "כ", "K": "כ", "Q": "ק",
    "L": "ל", "M": "מ", "N": "נ",
    "S": "ס", "T": "ת", "R": "ר", "P": "פ", "F": "פ",
    "J": "ג", "X": "קס"  # X approximated as kuf-samekh
}

def eng_to_heb(word: str) -> str:
    w = re.sub(r"[^A-Za-z]", "", word).upper()
    i, out = 0, []
    while i < len(w):
        if i + 1 < len(w):
            dig = w[i:i+2]
            if dig in DIGRAPHS:
                out.append(DIGRAPHS[dig])
                i += 2
                continue
        ch = w[i]
        mapped = SINGLE.get(ch, "")
        if mapped:
            out.append(mapped)
        i += 1
    return "".join(out)

# -----------------------------
# Data fetching and build
# -----------------------------
@st.cache_data(show_spinner=False)
def get_chapter_count(book: str) -> int:
    url = INDEX_API.format(book=quote(book))
    r = requests.get(url, timeout=30)
    r.raise_for_status()
    meta = r.json()
    lens = meta.get("schema", {}).get("lengths", [])
    return int(lens[0]) if lens else 0

@st.cache_data(show_spinner=False)
def fetch_book_hebrew(book: str) -> list[dict]:
    rows = []
    ch_count = get_chapter_count(book)
    for ch in range(1, ch_count + 1):
        ref = f"{book} {ch}"
        url = TEXTS_API.format(ref=quote(ref))
        r = requests.get(url, timeout=30)
        r.raise_for_status()
        data = r.json()
        verses = data.get("he", [])  # list of verse strings
        for v_idx, verse in enumerate(verses, start=1):
            clean = normalize_hebrew(verse if isinstance(verse, str) else " ".join(verse))
            rows.append({"book": book, "chapter": ch, "verse": v_idx, "text": clean})
        time.sleep(0.1)  # be gentle
    return rows

@st.cache_data(show_spinner=False)
def build_letters_and_refs() -> tuple[list[str], list[tuple[str,int,int]], dict]:
    verses = []
    for b in BOOKS:
        verses.extend(fetch_book_hebrew(b))
    letters: list[str] = []
    refs: list[tuple[str,int,int]] = []
    for row in verses:
        for ch in row["text"]:
            letters.append(ch)
            refs.append((row["book"], row["chapter"], row["verse"]))
    # Precompute positions for fast search
    positions: dict[str, list[int]] = {}
    for i, ch in enumerate(letters):
        positions.setdefault(ch, []).append(i)
    return letters, refs, positions

# -----------------------------
# ELS search
# -----------------------------
def find_els(
    letters: list[str],
    refs: list[tuple[str,int,int]],
    positions: dict,
    pattern: str,
    min_skip: int,
    max_skip: int,
    search_forward: bool,
    search_backward: bool,
    max_hits: int
) -> list[dict]:
    results: list[dict] = []
    n = len(letters)
    m = len(pattern)
    if m == 0:
        return results

    start_candidates = positions.get(pattern[0], [])
    for skip in range(min_skip, max_skip + 1):
        # Forward
        if search_forward:
            for start in start_candidates:
                end_idx = start + (m - 1) * skip
                if end_idx >= n:
                    continue
                ok = True
                for j in range(1, m):
                    if letters[start + j * skip] != pattern[j]:
                        ok = False
                        break
                if ok:
                    results.append(make_hit(letters, refs, start, skip, m))
                    if len(results) >= max_hits:
                        return results
        # Backward
        if search_backward:
            for start in start_candidates:
                end_idx = start - (m - 1) * skip
                if end_idx < 0:
                    continue
                ok = True
                for j in range(1, m):
                    if letters[start - j * skip] != pattern[j]:
                        ok = False
                        break
                if ok:
                    results.append(make_hit(letters, refs, start, -skip, m))
                    if len(results) >= max_hits:
                        return results
    return results

def make_hit(letters, refs, start, step, m) -> dict:
    idxs = [start + j * step for j in range(m)]
    first = min(idxs)
    last = max(idxs)
    # Context window in the linear stream
    pad = 40
    c_start = max(0, first - pad)
    c_end = min(len(letters), last + pad + 1)
    html = render_context(letters, idxs, c_start, c_end)
    book, ch, vs = refs[start]
    return {
        "book": book, "chapter": ch, "verse": vs,
        "skip": step,
        "index_start": start,
        "context_html": html,
        "match_indices": idxs
    }

def render_context(letters: list[str], match_idxs: list[int], c_start: int, c_end: int) -> str:
    match_set = set(match_idxs)
    spans = []
    for i in range(c_start, c_end):
        ch = letters[i]
        if i in match_set:
            spans.append(f'<span style="color:#fff;background:#e63946;border-radius:2px;padding:1px 2px;margin:0 0.5px">{ch}</span>')
        else:
            spans.append(f'<span style="color:#1f2937;margin:0 0.5px">{ch}</span>')
    return f'<div style="font-size:22px;line-height:1.8;direction:rtl;text-align:right;word-break:break-all">{"" .join(spans)}</div>'

def results_to_csv(results: list[dict], pattern: str) -> bytes:
    buf = io.StringIO()
    w = csv.writer(buf)
    w.writerow(["book","chapter","verse","skip","index_start","pattern_length"])
    for r in results:
        w.writerow([r["book"], r["chapter"], r["verse"], r["skip"], r["index_start"], len(r["match_indices"])])
    return buf.getvalue().encode("utf-8")

# -----------------------------
# UI
# -----------------------------
st.title("🔍 Torah Matrix (ELS)")
st.write("Type a word or name in English. We’ll transliterate it to Hebrew and search the Torah as an Equidistant Letter Sequence across your chosen skips.")

with st.sidebar:
    st.header("Search settings")
    search_word = st.text_input("Word or name (English)", value="Miles")
    min_skip, max_skip = st.slider("Skip range", 1, 2000, (1, 300))
    col_dir1, col_dir2 = st.columns(2)
    with col_dir1:
        forward = st.checkbox("Forward", value=True)
    with col_dir2:
        backward = st.checkbox("Backward", value=True)
    max_hits = st.number_input("Maximum results", min_value=10, max_value=5000, value=300, step=10)
    run_btn = st.button("Search", type="primary")

# Prepare data (cached)
with st.status("Preparing Hebrew text (first run may take a minute)...", expanded=False) as status:
    letters, refs, positions = build_letters_and_refs()
    status.update(label="Hebrew text ready", state="complete")

colA, colB = st.columns([1,2], gap="large")

with colA:
    heb = eng_to_heb(search_word or "")
    st.markdown("**Transliterated Hebrew pattern:**")
    st.code(heb if heb else "(no mappable letters)", language=None)
    st.markdown("**Books searched:** " + ", ".join(BOOKS))
    st.markdown("Tips:")
    st.markdown("- Shorter patterns hit more often; widen skip range gradually.")
    st.markdown("- Try swapping S for SH, TS for TZ, CH for KH if the sound differs.")

with colB:
    if run_btn:
        if not heb:
            st.error("Could not map any letters from that input to Hebrew.")
        elif not (forward or backward):
            st.error("Select at least one direction (Forward/Backward).")
        else:
            with st.spinner("Searching ELS patterns..."):
                hits = find_els(
                    letters=letters,
                    refs=refs,
                    positions=positions,
                    pattern=heb,
                    min_skip=min_skip,
                    max_skip=max_skip,
                    search_forward=forward,
                    search_backward=backward,
                    max_hits=int(max_hits),
                )
            if hits:
                st.success(f"Found {len(hits)} matches")
                dl = results_to_csv(hits, heb)
                st.download_button("Download results as CSV", data=dl, file_name="torah_matrix_results.csv", mime="text/csv")
                # Show results
                for i, r in enumerate(hits, start=1):
                    st.markdown(f"### {i}. {r['book']} {r['chapter']}:{r['verse']} — skip {r['skip']}")
                    st.markdown(r["context_html"], unsafe_allow_html=True)
                    st.divider()
            else:
                st.warning("No matches found in this skip range. Try a wider range or a shorter pattern.")
    else:
        st.info("Set your word and settings in the sidebar, then click Search.")
