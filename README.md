# Cendo – automatisk analyse af viderestillinger

Java-pipeline, der grupperer viderestillede samtaler i klynger og navngiver dem. Modellerne kan køre lokalt via **Ollama** (gratis) eller via **Hugging Face's API** (kræver credits). Det vælges i `config.properties`.

```
Transskription → LLM: primær årsag + forklaring → Embedding af den primære årsag → UMAP → HDBSCAN → LLM: klyngenavn → Resultater
```

| Trin | Hvad | Hvor |
|---|---|---|
| 1 | Indlæs viderestillede samtaler fra fanen *Model Input* | `data/DatasetLoader` |
| 2 | LLM giver en kort primær årsag (2-6 ord) og én forklarende sætning (kun pipeline 1). Kun den primære årsag embeddes og clustres; sætningen vises ved siden af i resultaterne | `extraction/LlmReasonExtractor` |
| 3 | Embeddings med bge-m3 | `embedding/OllamaEmbedder` / `HuggingFaceEmbedder` |
| 4 | UMAP reducerer til 5 dimensioner | `reduction/UmapReducer` |
| 5 | HDBSCAN finder klyngerne | `clustering/HdbscanClusterer` |
| 6 | LLM navngiver hver klynge | `naming/LlmClusterNamer` |
| 7 | Sammenligning med facit (fanen *Labels*) | `evaluation/ClusterEvaluator` |

## Opsætning (Ollama, standard)

1. **Java 25** (tjek under *File → Project Structure → SDK*).
2. **Installer Ollama** fra ollama.com. Den kører i baggrunden (lama-ikonet i proceslinjen).
3. **Hent modellerne** i en terminal:
   ```
   ollama pull qwen2.5:7b
   ollama pull bge-m3
   ```
4. Læg datasættet i mappen `data/`.
5. Kør `Main` i IntelliJ.

Første kørsel tager tid, fordi LLM'en kører på jeres egen computer. Alle svar gemmes i `cache/`, så næste kørsel er hurtig.

### Skift til Hugging Face's API

Sæt `llm.provider=HUGGINGFACE` og/eller `embedding.provider=HUGGINGFACE` i `config.properties`. Det kræver miljøvariablen `HF_TOKEN` (Fine-grained token med presettet *Inference*) og credits på Hugging Face-kontoen. Man kan godt blande, fx LLM via API og embeddings lokalt.

## Indstillinger (`config.properties`)

- `pipeline.mode`: `REASON_FIRST` (pipeline 1) eller `FULL_TRANSCRIPT` (pipeline 2). Skift og kør igen for at sammenligne.
- `llm.provider` / `embedding.provider`: `OLLAMA` eller `HUGGINGFACE`.
- `ollama.llm.model`: den lokale LLM. `qwen2.5:7b` er svag på dansk; prøv gerne en anden, fx `gemma3:12b` (kræver mere RAM). Husk `ollama pull <model>` først.
- `dataset.split`: `train`, `validation`, `test` eller `all`. Datasættet anbefaler at tune på train/validation og kun køre test én gang til sidst.
- `hdbscan.minPoints` og `hdbscan.minClusterSize`: de to HDBSCAN-parametre.
- `umap.enabled`: slå dimensionsreduktion til/fra.

## Sammenlign flere LLM'er

Pipelinen kan køre flere Ollama-modeller efter hinanden med præcis de samme indstillinger, så kun LLM'en skifter.

1. Hent modellerne:
   ```
   ollama pull gemma3:12b
   ollama pull llama3.1:8b
   ollama pull mistral-nemo
   ollama pull hf.co/danish-foundation-models/DFM-Mimir-v1.5-GGUF:Q4_K_M
   ```
2. I IntelliJ: *Edit Configurations* → *Main* → skriv `config-sammenligning.properties` i feltet **Program arguments**.
3. Kør `Main`.

`config-sammenligning.properties` bruger de 50 omskrevne samtaler i `data/samtaler_50_40_viderestillinger_pipeline.xlsx` (40 viderestillinger) og lister modellerne i `compare.models`. Indstillingen findes også i `config.properties`; står den tom, kører pipelinen som før med `ollama.llm.model`.

Resultater:
- `results/sammenligning_<split>.csv`: én linje pr. model med klynger, støj, ARI, NMI og LLM-tid pr. samtale (åbner i Excel).
- `results/<pipeline>_<split>_<model>/`: de normale resultatfiler for hver model.

Tabellen har også to mål, der ikke afhænger af UMAP og HDBSCAN, og som derfor er de mest pålidelige på små datasæt:
- **Nærmeste nabo**: andelen af samtaler, hvor den mest lignende årsag (cosinus på de fulde embeddings) har samme facit.
- **ARI(k)/NMI(k)**: average-linkage-clustering med præcis lige så mange klynger, som der er sande årsager. NMI er generelt høj, når der er mange små klynger, så ARI(k) er det bedste af de to til at sammenligne.

Hvis en model fejler (fx fordi den ikke er hentet), skrives fejlen i tabellen, og de andre modeller kører videre. Tiden pr. samtale er kun retvisende første gang en model kører, for derefter kommer svarene fra cachen.

## Cache

Alle API-svar gemmes i `cache/`. Kører I pipelinen igen med nye HDBSCAN- eller UMAP-parametre, bliver der ikke kaldt API'et igen, og det er både hurtigt og gratis. Slet `cache/` for at starte forfra.

## Resultater

For hver kørsel skrives til `results/<pipeline>_<split>/`:
- `summary.txt`: klynger med navne, eksempler og evaluering
- `assignments.csv`: én linje pr. samtale (åbner i Excel)
- `clusters.json`: klynger med medlemmer, et godt udgangspunkt for dashboardet

### Evaluering
Pipelinen sammenligner klyngerne med facit i `transfer_classification`:
- **ARI** og **NMI**: 1,0 = samme gruppering som facit, 0 = ikke bedre end tilfældigt.
- **Andel støj**: hvor mange samtaler HDBSCAN ikke placerede i en klynge.
- **Renhed pr. klynge**: hvor stor en del af klyngen der har samme sande årsag.

Facit bruges først *efter* clustering og aldrig som input til modellerne.

## Vigtigt: datalækage

Transskriptionerne indeholder agentens tool-kald med facit, fx `TOOL: transfer({reason: "price_or_invoice"})`. `TranscriptSanitizer` fjerner argumenterne (`TOOL: transfer()`), før teksten sendes til modellerne. Ellers ville klyngerne bare kopiere facit.

## Designmønstre (til rapporten)

- **Strategy**: hvert trin er et interface (`ChatClient`, `ReasonExtractor`, `Embedder`, `DimensionReducer`, `Clusterer`, `ClusterNamer`), så implementeringer kan udskiftes uden at ændre resten.
- **Adapter**: `OllamaChatClient`, `HuggingFaceChatClient`, `OllamaEmbedder`, `HuggingFaceEmbedder`, `HdbscanClusterer` og `UmapReducer` oversætter eksterne API'er/biblioteker til vores interfaces.
- **Factory**: `ProviderFactory` vælger Ollama- eller Hugging Face-implementeringen ud fra `config.properties`. Det er det eneste sted i koden, der kender forskel på udbyderne.
- **Decorator**: `CachingReasonExtractor` og `CachingEmbedder` tilføjer caching til et andet objekt med samme interface.

## Fejlfinding

**"Kunne ikke forbinde til localhost:11434".** Ollama kører ikke. Start programmet Ollama, og tjek at `http://localhost:11434` viser *Ollama is running*.

**HTTP 404 fra Ollama.** Modellen er ikke hentet. Kør `ollama pull <modelnavn>`.

**UMAP kan ikke køre (`UnsatisfiedLinkError`).** Smiles UMAP kræver de native biblioteker OpenBLAS og ARPACK. Pipelinen fortsætter automatisk uden UMAP, men resultatet bliver typisk dårligere.
- Windows: hent Smile-release-pakken fra GitHub og tilføj dens `bin`-mappe til `PATH`.
- Mac: `brew install arpack` og kopier `/opt/homebrew/lib/libarpack.dylib` til projektmappen.
- Linux: `sudo apt install libopenblas-dev libarpack2`.

**HTTP 404/400 fra Hugging Face.** Modellen er måske ikke tilgængelig via Inference Providers lige nu. Tjek modellens side på huggingface.co under *Inference Providers*, og skift model i `config.properties`. Et alternativ til embeddings er `intfloat/multilingual-e5-large`.

**HTTP 402 eller 429 fra Hugging Face.** Den gratis kvote er brugt, eller I sender for mange kald. Vent, eller tjek kvoten under *Settings → Billing*. Cachen sikrer, at allerede hentede svar ikke går tabt.

**Alle samtaler havner i én klynge.** Prøv med UMAP slået til, lavere `minClusterSize` eller et større split.
