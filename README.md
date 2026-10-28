# Multimodal AI Workspace pentru Documente și Imagini

Lucrare de licență, Automatică și Calculatoare, Universitatea Politehnica Timișoara.

## Ideea pe scurt

Dezvolt un spațiu de lucru inteligent în care utilizatorul încarcă **documente (PDF), capturi de ecran, diagrame și imagini** și interacționează cu ele în limbaj natural. Sistemul combină un **Vision-Language Model (VLM)**, pentru conținut vizual, cu **Retrieval-Augmented Generation (RAG)**, pentru texte lungi. Poate să răspundă la întrebări cu dovezi din fișiere, să extragă informații structurate, să compare surse vizuale și text și să genereze rezumate.

**AI-ul folosit:** Claude (prin API-ul Anthropic), pentru înțelegere vizuală, răspunsuri și extracție. Nu antrenez niciun model. Accentul este pe o aplicație fiabilă și pe o evaluare corectă a deciziilor de design.

**Cerințe importante:**
- MVP-ul cere: upload și gestiune de fișiere, PDF + imagini + capturi, extracție de text și chunking, Q&A ancorat în fișiere, interpretare VLM pentru diagrame, extracție structurată pentru cel puțin un caz de utilizare și mod de comparare pentru două surse.
- Aplicația **nu trebuie să ruleze pe localhost**, deci o deployez online.

## Cuprins

1. [Problema](#1-problema)
2. [Documentare și state of the art](#2-documentare-și-state-of-the-art)
3. [Proiectare](#3-proiectare)
4. [Implementare](#4-implementare)
5. [Testare și evaluare](#5-testare-și-evaluare)
6. [Concluzii](#6-concluzii)
7. [Obiective și funcționalități](#7-obiective-și-funcționalități)

---

## 1. Problema

### Ce problemă rezolv?

Documentele reale amestecă text, tabele, capturi de UI și diagrame. Uneltele obișnuite le tratează slab:

- chatbot-urile doar pe text pierd informația din diagrame și imagini (iar OCR-ul pierde layout-ul);
- modelele vizuale înțeleg imaginile, dar nu pot procesa singure documente foarte lungi (limite de context, cost, latență);
- verificarea „captura de ecran respectă cerințele scrise?” se face manual, lent și cu erori.

### Utilizatori țintă

- **studenți și cercetători:** întrebări peste mai multe lucrări, date din figuri și tabele;
- **ingineri software, analiști, QA:** cerințe din specificații, comparare UI cu cerințele;
- **echipe mici:** rezumate de rapoarte, compararea a două versiuni de document.

### Scenariu de demonstrație

Utilizatorul încarcă o specificație PDF, o captură de UI și o diagramă de arhitectură. Aplicația:

1. rezumă specificația;
2. răspunde la o întrebare cu dovadă la nivel de pagină;
3. explică diagrama;
4. extrage cerințele într-o listă structurată;
5. compară captura de ecran cu cerințele scrise.

### Ce nu fac

Nu antrenez OCR sau modele vizuale proprii, nu suport toate formatele office, nu construiesc un editor colaborativ și nu mă bazez pe text liber de la model atunci când am nevoie de date structurate.

### Ce înseamnă „extracție structurată”

Pe scurt: **îi cer modelului să îmi răspundă ca un formular completat, nu ca un eseu.**

**Exemplu.** Am o specificație de 30 de pagini și vreau cerințele din ea.

- **Răspuns liber (ce face un chatbot obișnuit):** „În document se spune că utilizatorul trebuie să se poată autentifica, iar pe pagina 5 apare și ceva despre export...”. Poate fi corect, dar formatul se schimbă la fiecare întrebare și programul meu nu poate lua din el, în mod sigur, „care e cerința 1” sau „pe ce pagină e”.
- **Răspuns structurat (ce vreau eu):** modelul completează un tabel cu coloane fixe, de exemplu:

| id | text | prioritate | pagină |
|---|---|---|---|
| R1 | Utilizatorul se poate autentifica | mare | 3 |
| R2 | Utilizatorul poate exporta datele | mică | 5 |

Coloanele (id, text, prioritate, pagină) formează **schema**: lista de câmpuri pe care i le impun. Tehnic, răspunsul vine în format JSON, adică o formă de text pe care programele o citesc ușor.

**De ce îmi trebuie:**
- pot afișa rezultatul direct într-un tabel în aplicație;
- pot verifica automat dacă e corect completat (de exemplu, „pagina chiar există în document?”). Dacă nu e, cer modelului să încerce din nou;
- pot compara cerințele cu captura de ecran, punct cu punct, ceea ce din text liber nu ar merge fiabil.

---

## 2. Documentare și state of the art

### 2.1 Aplicații care rezolvă parțial problema

Scop: să arăt în lucrare ce există și ce lipsește, ca să justific proiectul. Le voi testa pe aceleași 3 fișiere (PDF, captură, diagramă).

| Resursă | De ce o citesc / testez | Link |
|---|---|---|
| **Google NotebookLM** | Cea mai apropiată idee: răspunsuri cu citări către sursă. Văd ce face bine la UX și ce îi lipsește (scheme de extracție proprii, comparare de pipeline-uri). | ✔ https://notebooklm.google.com |
| **Adobe Acrobat AI Assistant** | Asistent integrat în PDF, cu citări. Verific cât de bine tratează imagini și grafice, ca să mă compar. | ✔ https://www.cnbc.com/2024/02/20/adobe-launches-ai-assistant-that-can-search-and-summarize-pdfs.html |
| **AnythingLLM** | Alternativă open-source „chat cu documentele tale”, cu workspace-uri. Mă ajută la ideea de workspace și arată limitele unui sistem orientat pe text. | ✔ https://github.com/Mintplex-Labs/anything-llm |
| **RAGFlow** | Motor RAG open-source cu citări trasabile și parsare de layout. Îl studiez ca model de pipeline și de afișare a dovezilor. | ✔ https://github.com/infiniflow/ragflow |
| **Claude.ai (încărcare de fișiere)** | Tipul de chat general la care mă compar: nu are workspace persistent propriu și nici extracție validată programatic. | |

**Cum mă diferențiez:**
1. aleg automat, pe fiecare pagină, între procesare text și procesare vizuală;
2. extracție cu scheme explicite, validată programatic;
3. mod de comparare între două surse;
4. dovezi (fișier și pagină) în fiecare răspuns;
5. evaluare reproductibilă VLM direct vs. OCR + LLM;
6. aplicație deployată online.

### 2.2 Fundamente: RAG și evaluarea lui

Scop: să înțeleg teoria din spatele părții de „text lung” și să știu cum măsor calitatea.

| Resursă | De ce o citesc | Link |
|---|---|---|
| **Lewis et al. (2020), Retrieval-Augmented Generation for Knowledge-Intensive NLP Tasks** | Lucrarea care a definit RAG. O citesc ca să pot explica în capitolul 2 de ce folosesc retrieval în loc să trimit tot documentul modelului. | ✔ https://arxiv.org/abs/2005.11401 |
| **RAGAS** | Definește metricile faithfulness și answer relevance. Le folosesc la evaluare pentru a măsura cât de ancorate sunt răspunsurile în fișiere. | ✔ https://arxiv.org/abs/2309.15217 |
| **ARES** | Alternativă la RAGAS, cu judecători LLM adaptați. Util pentru metodologie (pot justifica de ce aleg o metodă sau alta). | ✔ https://arxiv.org/abs/2311.09476 |
| **RAGAS pe GitHub** | Cod și documentație, ca să îl rulez efectiv în scriptul de evaluare. | ✔ https://github.com/explodinggradients/ragas |

### 2.3 OCR vs. VLM

Scop: experimentul meu principal compară „VLM direct” cu „OCR + LLM”. Aceste lucrări îmi spun ce să mă aștept și cum să proiectez comparația.

| Resursă | De ce o citesc | Link |
|---|---|---|
| **MMLongBench-Doc (NeurIPS 2024)** | Cel mai important pentru mine. Arată că pe documente lungi multe modele vizuale stau mai prost decât un LLM alimentat cu text extras prin OCR. Include și întrebări fără răspuns (pentru halucinații), idee pe care o copiez în benchmark-ul meu. | ✔ https://arxiv.org/abs/2407.01523 |
| **Vision-Based Approaches to Robust Document Retrieval (2025)** | Compară un RAG care lucrează direct cu imaginile paginilor cu RAG clasic bazat pe OCR, la calități diferite ale documentelor. Model pentru experimentele mele de comparare. | ✔ https://arxiv.org/abs/2505.05666 |
| **Survey: Deep Learning based Visually Rich Document Content Understanding** | Sinteză OCR-based vs. OCR-free; mă ajută la capitolul de state of the art. | ✔ https://arxiv.org/abs/2408.01287 |
| **Survey: MLLM-based Visually Rich Document Understanding** | Versiune mai nouă a aceleiași teme; arată cum se apropie modelele OCR-free de cele bazate pe OCR. | ✔ https://arxiv.org/abs/2507.09861 |
| **LayoutLMv3** | Reper clasic pentru abordarea bazată pe OCR. Îl citesc pentru context, nu îl implementez. | ✔ https://arxiv.org/abs/2204.08387 |
| **Donut (OCR-free Document Understanding Transformer)** | Prima abordare influentă fără OCR. Context pentru „de ce există VLM-uri pe documente”. | ✔ https://arxiv.org/abs/2111.15664 |
| **Tesseract** | Motorul OCR pe care îl folosesc ca bază în varianta „OCR + LLM”. | ✔ https://github.com/tesseract-ocr/tesseract |

### 2.4 Parsarea documentelor (partea fără AI din pipeline)

Scop: au legătură directă cu implementarea (aplicația fără AI).

| Resursă | De ce o citesc | Link |
|---|---|---|
| **Docling (IBM)** | Convertește PDF/imagini în Markdown/JSON cu layout și tabele. Candidat principal pentru extracția de text și chunking. | ✔ https://github.com/DS4SD/docling |
| **Docling Technical Report** | Înțeleg ce produce exact parsarea (ierarhie, număr de pagină), util ca să păstrez referințele la pagină. | ✔ https://arxiv.org/abs/2408.09869 |
| **PyMuPDF** | Randarea paginilor PDF ca imagini, necesară pentru a le trimite către VLM. | ✔ https://pymupdf.readthedocs.io |
| **pdfplumber** | Extracție de text și tabele din PDF; variantă mai simplă decât Docling. | ✔ https://github.com/jsvine/pdfplumber |

### 2.5 Claude API (AI-ul din aplicație)

Scop: sunt resursele pe care le voi folosi cel mai mult în implementare.

| Resursă | De ce o citesc | Link |
|---|---|---|
| **Claude: PDF support** | Explică cum primește Claude un PDF (text extras plus imaginea fiecărei pagini), limitele de pagini/dimensiune și costul în tokeni. Decide cum trimit documentele. Limitele diferă între versiuni, le reverific. | ✔ https://platform.claude.com/docs/en/build-with-claude/pdf-support |
| **Claude: Citations** | Dă referințe native la pagină pentru PDF-uri (indexate de la 1). Acoperă cerința de dovezi. Vezi nota de mai jos despre incompatibilitatea cu Structured Outputs. | ✔ https://platform.claude.com/docs/en/build-with-claude/citations |
| **Claude: cookbook pentru citări** | Cod de exemplu pentru citări; economisește timp. | ✔ https://platform.claude.com/cookbook/misc-using-citations |
| **Claude: Vision** | Cum trimit capturi de ecran și diagrame ca imagini (formate, dimensiuni, costuri). Partea de VLM din aplicație. | ✔ https://platform.claude.com/docs/en/build-with-claude/vision |
| **Claude: Structured Outputs** | Cum obțin JSON care respectă o schemă. Baza pentru extracția de cerințe validată programatic. | ✔ https://platform.claude.com/docs/en/build-with-claude/structured-outputs |
| **Anthropic Python SDK** | Biblioteca pe care o folosesc din backend-ul Python. | ✔ https://github.com/anthropics/anthropic-sdk-python |

**Notă de design (citări vs. extracție structurată):** documentația Claude spune că **Citations** și **Structured Outputs** nu pot fi folosite în același apel (cererea returnează eroare 400). Sunt două funcții bune, dar separate:
- *Citations*: răspuns în text, cu trimiteri exacte la pagină.
- *Structured Outputs*: răspuns strict în JSON, conform schemei.

Soluții posibile, de decis la proiectare:
1. fac **două apeluri**: unul pentru răspunsul cu citări (Q&A), altul pentru extracția structurată;
2. pentru extracție, pun în schemă câmpul `source_page`, iar codul meu **verifică** după aceea că pagina există și că textul extras apare pe ea.

### 2.6 Tehnologii pe care le folosesc

- **Frontend:** React + TypeScript
- **Backend:** Python + FastAPI
- **Bază de date:** PostgreSQL + pgvector (metadate și embeddings)
- **Parsare documente:** Docling, PyMuPDF
- **OCR (pentru comparație):** Tesseract
- **AI:** Claude API, prin SDK-ul Python Anthropic
- **Validarea ieșirilor structurate:** Pydantic
- **Teste:** pytest
- **Deployment:** Docker + Docker Compose pe un server în cloud, cu GitHub Actions pentru testare și deploy

---

## 3. Proiectare

### Arhitectura (schiță inițială)

| Componentă | Responsabilitate |
|---|---|
| Client web | Încărcare fișiere, previzualizare, chat, extracție, comparare. |
| Backend / API | Autentificare, fișiere, metadate, persistență, orchestrarea apelurilor către Claude. |
| Pipeline documente | Extracție text, chunking, embeddings, căutare vectorială. |
| Strat multimodal | Claude (vision) pentru capturi, diagrame și pagini complexe vizual. |
| Strat de ancorare | Combină textul recuperat cu observațiile vizuale înainte de generare. |
| Validare | Verifică ieșirile structurate față de scheme (Pydantic); reîncearcă la eroare. |

### Cum integrez AI

- **Routing pe pagină:** pagină cu mult text se extrage ca text; pagină vizuală sau cu OCR slab se trimite ca imagine către Claude.
- **RAG** pentru documente lungi: chunking, embeddings, căutare, răspuns cu dovezi.
- **Ancorare:** modelul răspunde doar din dovezile primite și spune „nu am găsit” în rest.
- **Extracție structurată** cu scheme explicite și validare programatică.
- **Comparare:** extrag fapte din ambele surse și le potrivesc una câte una (satisfăcută / contrazisă / nevăzută).

### Tehnologii propuse (de confirmat)

React + TypeScript, Python + FastAPI, PostgreSQL + pgvector, Docling / PyMuPDF, Tesseract (bază OCR), Claude API (model principal), Pydantic, Docker pe un server în cloud, GitHub Actions. Cheile API rămân doar pe server.

### Diagrame de realizat

- [ ] use case
- [ ] user flow
- [ ] mockup-uri UX (Figma)
- [ ] arhitectură
- [ ] secvență (Q&A, extracție)
- [ ] model de date (ER)

---

## 4. Implementare

De completat pe parcurs, începând cu aplicația fără AI și apoi cu partea de AI.

---

## 5. Testare și evaluare

### Testare software

- teste unitare pentru logica deterministă (chunking, routing, validare scheme, calcul de cost);
- teste de integrare pentru fluxul principal: încărcare, întrebare, răspuns cu dovezi.

### Performanță

Latență end-to-end, timp de indexare pe pagină, cost aproximativ per sarcină (loghez tokenii fiecărui apel).

### Experimente AI

Folosesc un benchmark mic și fix (documente publice sau create de mine), cu răspunsuri așteptate scrise manual, inclusiv câteva întrebări fără răspuns în document, pentru a măsura halucinațiile.

Experimente pe care vreau să le fac:

| Experiment | Metrici |
|---|---|
| VLM direct vs. OCR + LLM | acuratețe, latență, cost |
| Cel puțin două modele sau pipeline-uri comparate pe aceleași intrări (ex. două modele Claude diferite, plus opțional alt furnizor) | acuratețe, latență, cost |
| RAG vs. context complet trimis modelului (opțional) | acuratețe, latență, cost |

---

## 6. Concluzii

De completat la final: ce funcționează, limite (erori OCR, halucinații, cost, confidențialitatea fișierelor, dependența de API-uri externe, benchmark mic), ce am învățat și dezvoltări viitoare.

---

## 7. Obiective și funcționalități

### Funcționalități de bază (MVP)

- [ ] workspace cu încărcare și gestionare de fișiere (PDF, imagini, capturi)
- [ ] extracție de text și chunking
- [ ] întrebări în limbaj natural, cu dovezi
- [ ] interpretare VLM pentru diagrame și pagini vizuale
- [ ] extracție structurată pentru un caz de utilizare (ex. cerințe)
- [ ] comparare a două documente/imagini
- [ ] deployment la un URL public

### Funcționalități extra (doar după MVP)

- [ ] detectarea contradicțiilor între documente
- [ ] clasificare și etichetare automată
- [ ] citări vizuale (regiuni evidențiate)
- [ ] procesare locală/privată pentru fișiere selectate

### Livrabile finale

- aplicație funcțională cu flux complet și repository curat, cu instrucțiuni de rulare;
- diagramă de arhitectură și explicarea deciziilor;
- script/notebook de evaluare și set de test reproductibil, cu metrici;
- lucrarea, cu distincție clară între contribuția mea și serviciile AI externe;
- demo live fiabil plus un demo înregistrat de rezervă.
