# 🧪 **Legal Concept Distillation - Active Learning Loop Avanzato**

## 🎯 **Vision Strategica**

Creare un sistema che **impara continuamente** dai casi ambigui, migliorando la comprensione dei concetti giuridici attraverso un ciclo virtuoso di:
1. **Identificazione automatica** delle lacune di conoscenza
2. **Campionamento intelligente** dei casi più informativi
3. **Annotazione guidata** da esperti legali
4. **Refinement incrementale** del modello
5. **Validazione e deployment** automatizzato

---

## 📐 **Architettura Multi-Layer**

```
┌─────────────────────────────────────────────────────────────────┐
│            LEGAL CONCEPT DISTILLATION PIPELINE                  │
├─────────────────────────────────────────────────────────────────┤
│                                                                 │
│  LAYER 1: UNCERTAINTY DETECTION                                 │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ • Bayesian Neural Networks (dropout-based uncertainty)   │  │
│  │ • Ensemble disagreement (3+ models vote)                 │  │
│  │ • Calibrated confidence scores (Platt scaling)           │  │
│  │ • Semantic entropy in embedding space                    │  │
│  └───────────────────────────────────────────────────────────┘  │
│                           ↓                                     │
│  LAYER 2: ACTIVE SAMPLING STRATEGY                              │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ • Uncertainty sampling (highest entropy)                 │  │
│  │ • Query-by-committee (max disagreement)                  │  │
│  │ • Diversity sampling (coverage in concept space)         │  │
│  │ • Expected model change (gradient-based)                 │  │
│  └───────────────────────────────────────────────────────────┘  │
│                           ↓                                     │
│  LAYER 3: HUMAN-IN-THE-LOOP ANNOTATION                          │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ • Guided annotation interface (pre-filled suggestions)   │  │
│  │ • Multi-annotator consensus (3 lawyers, majority vote)   │  │
│  │ • Quality control (gold standard checks)                 │  │
│  │ • Iterative refinement (annotator feedback loop)         │  │
│  └───────────────────────────────────────────────────────────┘  │
│                           ↓                                     │
│  LAYER 4: KNOWLEDGE DISTILLATION                                │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ • Teacher-Student framework (Opus → Sonnet → Haiku)      │  │
│  │ • Contrastive learning (hard negatives mining)           │  │
│  │ • Few-shot prompt optimization (DSPy framework)          │  │
│  │ • Continual learning (avoid catastrophic forgetting)     │  │
│  └───────────────────────────────────────────────────────────┘  │
│                           ↓                                     │
│  LAYER 5: VALIDATION & DEPLOYMENT                               │
│  ┌───────────────────────────────────────────────────────────┐  │
│  │ • A/B testing (canary deployment)                        │  │
│  │ • Regression testing (gold set performance)              │  │
│  │ • Monitoring (data drift detection)                      │  │
│  │ • Automated rollback (if metrics degrade)                │  │
│  └───────────────────────────────────────────────────────────┘  │
│                                                                 │
└─────────────────────────────────────────────────────────────────┘
```

---

## 🔬 **Technology Stack Dettagliato**

### **1. Uncertainty Quantification**

#### **A) Monte Carlo Dropout (Bayesian Approximation)**

```python
import torch
import torch.nn as nn
from typing import Tuple

class BayesianLegalClassifier(nn.Module):
    """
    Classifier con dropout attivo in inference per stimare incertezza.
    
    Paper: "Dropout as a Bayesian Approximation" (Gal & Ghahramani, 2016)
    """
    
    def __init__(self, input_dim: int, num_classes: int, dropout_rate: float = 0.3):
        super().__init__()
        self.encoder = nn.Sequential(
            nn.Linear(input_dim, 512),
            nn.ReLU(),
            nn.Dropout(dropout_rate),  # Attivo anche in eval()
            nn.Linear(512, 256),
            nn.ReLU(),
            nn.Dropout(dropout_rate),
            nn.Linear(256, num_classes)
        )
        self.dropout_rate = dropout_rate
    
    def forward(self, x: torch.Tensor, mc_samples: int = 30) -> Tuple[torch.Tensor, torch.Tensor]:
        """
        Returns:
            predictions: Mean prediction across MC samples
            uncertainty: Predictive entropy
        """
        self.train()  # Enable dropout
        
        predictions = []
        for _ in range(mc_samples):
            logits = self.encoder(x)
            probs = torch.softmax(logits, dim=-1)
            predictions.append(probs)
        
        predictions = torch.stack(predictions)  # [mc_samples, batch, num_classes]
        
        # Predictive mean
        mean_pred = predictions.mean(dim=0)
        
        # Predictive entropy (epistemic uncertainty)
        entropy = -torch.sum(mean_pred * torch.log(mean_pred + 1e-10), dim=-1)
        
        return mean_pred, entropy

# Usage
model = BayesianLegalClassifier(input_dim=1536, num_classes=50)
predictions, uncertainty = model(embeddings, mc_samples=50)

# Flag low-confidence samples
low_confidence = uncertainty > threshold  # e.g., > 2.0 bits
```

#### **B) Ensemble Disagreement**

```python
from scipy.stats import entropy
from scipy.special import softmax

class EnsembleLegalOracle:
    """
    Ensemble di 3+ modelli con votazione e disagreement scoring.
    """
    
    def __init__(self, models: list):
        self.models = models  # [GPT-4o-mini, Gemini-1.5-Flash, Claude-Haiku]
    
    async def predict_with_disagreement(
        self, 
        text: str, 
        concept: str
    ) -> dict:
        """
        Returns:
            prediction: Majority vote
            disagreement: Jensen-Shannon divergence between models
            samples_needed: bool (trigger human annotation)
        """
        predictions = []
        
        for model in self.models:
            logits = await model.classify(text, concept)
            predictions.append(softmax(logits))
        
        predictions = np.array(predictions)  # [n_models, n_classes]
        
        # Majority vote
        votes = predictions.argmax(axis=1)
        majority = np.bincount(votes).argmax()
        
        # Jensen-Shannon divergence (symmetrized KL)
        mean_pred = predictions.mean(axis=0)
        disagreement = sum([
            entropy(pred, mean_pred) for pred in predictions
        ]) / len(predictions)
        
        # Trigger annotation if high disagreement
        samples_needed = disagreement > 0.5  # Threshold empirico
        
        return {
            "prediction": majority,
            "confidence": 1 - disagreement,
            "disagreement": disagreement,
            "needs_annotation": samples_needed,
            "votes": votes.tolist()
        }
```

#### **C) Semantic Entropy in Embedding Space**

```python
import numpy as np
from sklearn.neighbors import NearestNeighbors

class SemanticUncertaintyDetector:
    """
    Misura incertezza basata su densità locale nello spazio embedding.
    
    Intuition: Punti isolati = concetti poco rappresentati = alta incertezza
    """
    
    def __init__(self, embeddings: np.ndarray, k: int = 10):
        self.embeddings = embeddings
        self.nn = NearestNeighbors(n_neighbors=k, metric='cosine')
        self.nn.fit(embeddings)
        
        # Calcola densità locale (distanza media ai k vicini)
        distances, _ = self.nn.kneighbors(embeddings)
        self.local_density = distances.mean(axis=1)
    
    def predict_uncertainty(self, query_embedding: np.ndarray) -> float:
        """
        Returns uncertainty score [0, 1] basato su densità locale.
        """
        distances, indices = self.nn.kneighbors(query_embedding.reshape(1, -1))
        
        # Distanza media ai vicini (normalizzata)
        avg_distance = distances.mean()
        
        # Z-score rispetto alla distribuzione globale
        z_score = (avg_distance - self.local_density.mean()) / self.local_density.std()
        
        # Converti in probabilità [0, 1]
        uncertainty = 1 / (1 + np.exp(-z_score))
        
        return uncertainty

# Usage
detector = SemanticUncertaintyDetector(train_embeddings, k=15)
uncertainty = detector.predict_uncertainty(new_embedding)

if uncertainty > 0.7:
    trigger_annotation(document)
```

---

### **2. Active Sampling Strategies**

#### **A) Hybrid Sampling (Multi-Criteria)**

```python
from dataclasses import dataclass
import heapq

@dataclass
class SampleCandidate:
    doc_id: str
    uncertainty: float
    diversity_score: float
    expected_impact: float
    legal_importance: float  # NEW: Peso articoli centrali
    
    @property
    def composite_score(self) -> float:
        """
        Weighted combination of criteria.
        """
        return (
            0.35 * self.uncertainty +           # Priorità incertezza
            0.25 * self.diversity_score +       # Coverage spazio concetti
            0.20 * self.expected_impact +       # Impact su metriche
            0.20 * self.legal_importance        # Importanza giuridica
        )

class ActiveSampler:
    def __init__(self, budget: int = 100):
        self.budget = budget
        self.selected = []
    
    def sample_batch(
        self, 
        candidates: list[SampleCandidate],
        batch_size: int = 10
    ) -> list[str]:
        """
        Greedy selection con diversity constraint.
        """
        # Heap con top-K per composite score
        heap = [(-c.composite_score, c) for c in candidates]
        heapq.heapify(heap)
        
        selected = []
        selected_embeddings = []
        
        while len(selected) < batch_size and heap:
            _, candidate = heapq.heappop(heap)
            
            # Diversity check: skip if too similar to already selected
            if selected_embeddings:
                min_distance = min([
                    cosine_distance(candidate.embedding, emb)
                    for emb in selected_embeddings
                ])
                
                if min_distance < 0.1:  # Too similar
                    continue
            
            selected.append(candidate.doc_id)
            selected_embeddings.append(candidate.embedding)
        
        return selected
```

#### **B) Query-by-Committee con Legal Importance**

```python
class LegalQueryByCommittee:
    """
    QBC adattato per prioritizzare articoli centrali nel citation graph.
    """
    
    def __init__(self, citation_graph: nx.Graph):
        self.graph = citation_graph
        
        # PageRank giuridico (articoli più citati)
        self.pagerank = nx.pagerank(citation_graph)
    
    async def select_samples(
        self, 
        unlabeled_pool: list[Document],
        n_samples: int = 20
    ) -> list[Document]:
        """
        Combina disagreement + importanza giuridica.
        """
        scores = []
        
        for doc in unlabeled_pool:
            # Ensemble disagreement (come prima)
            disagreement = await self.ensemble.predict_disagreement(doc.text)
            
            # Legal importance (PageRank degli articoli citati)
            cited_articles = extract_citations(doc.text)
            importance = sum([
                self.pagerank.get(art, 0) for art in cited_articles
            ]) / (len(cited_articles) + 1)
            
            # Composite score
            score = 0.6 * disagreement + 0.4 * importance
            scores.append((score, doc))
        
        # Top-N
        scores.sort(reverse=True)
        return [doc for _, doc in scores[:n_samples]]
```

---

### **3. Human-in-the-Loop Annotation Platform**

#### **A) Annotation Interface (Streamlit-based)**

```python
import streamlit as st
from typing import Literal

class LegalAnnotationUI:
    """
    UI per annotazione guidata con pre-filled suggestions.
    """
    
    def render_annotation_task(self, doc: Document, suggestions: dict):
        st.title("🧑‍⚖️ Legal Concept Annotation")
        
        # Document context
        st.markdown(f"**URN:** `{doc.urn_nir}`")
        st.markdown(f"**Text:** {doc.text[:500]}...")
        
        # Pre-filled suggestions (da modello)
        st.subheader("🤖 Model Suggestions (confidence)")
        
        cols = st.columns(len(suggestions['concepts']))
        for i, (concept, conf) in enumerate(suggestions['concepts'].items()):
            with cols[i]:
                st.metric(concept, f"{conf:.2%}")
        
        # Human annotation
        st.subheader("✍️ Your Annotation")
        
        concept = st.selectbox(
            "Primary Legal Concept",
            options=LEGAL_CONCEPTS,
            index=suggestions['predicted_index']
        )
        
        confidence = st.slider(
            "Your confidence (0=guess, 100=certain)",
            0, 100, 80
        )
        
        # Additional context
        related_articles = st.multiselect(
            "Related Articles (optional)",
            options=doc.extracted_citations
        )
        
        notes = st.text_area("Notes / Edge Cases")
        
        # Submit
        if st.button("✅ Submit Annotation"):
            await self.save_annotation({
                "doc_id": doc.id,
                "concept": concept,
                "annotator_confidence": confidence / 100,
                "related_articles": related_articles,
                "notes": notes,
                "model_suggestions": suggestions,
                "timestamp": datetime.utcnow()
            })
            
            st.success("Saved! Next document loading...")
            st.rerun()
```

#### **B) Multi-Annotator Consensus**

```python
from scipy.stats import fleiss_kappa
from collections import Counter

class AnnotationAggregator:
    """
    Gestisce consensus tra annotatori multipli.
    """
    
    def __init__(self, min_annotators: int = 3, agreement_threshold: float = 0.7):
        self.min_annotators = min_annotators
        self.agreement_threshold = agreement_threshold
    
    async def aggregate_annotations(
        self, 
        doc_id: str
    ) -> dict:
        """
        Returns final label + inter-annotator agreement.
        """
        annotations = await self.db.get_annotations(doc_id)
        
        if len(annotations) < self.min_annotators:
            return {"status": "pending", "n_annotations": len(annotations)}
        
        # Majority vote
        labels = [a['concept'] for a in annotations]
        counts = Counter(labels)
        majority_label, majority_count = counts.most_common(1)[0]
        
        # Agreement rate
        agreement = majority_count / len(labels)
        
        # Fleiss' Kappa (inter-rater reliability)
        kappa = fleiss_kappa(self._format_for_kappa(annotations))
        
        # Confidence-weighted voting
        weighted_scores = defaultdict(float)
        for ann in annotations:
            weighted_scores[ann['concept']] += ann['annotator_confidence']
        
        weighted_label = max(weighted_scores, key=weighted_scores.get)
        
        # Final decision
        if agreement >= self.agreement_threshold and kappa > 0.6:
            final_label = majority_label
            quality = "high"
        elif weighted_label == majority_label:
            final_label = weighted_label
            quality = "medium"
        else:
            # Disagreement: escalate to senior annotator
            return {
                "status": "conflict",
                "labels": labels,
                "kappa": kappa,
                "needs_expert_review": True
            }
        
        return {
            "status": "completed",
            "label": final_label,
            "agreement": agreement,
            "kappa": kappa,
            "quality": quality,
            "n_annotations": len(annotations)
        }
```

---

### **4. Knowledge Distillation (Teacher-Student)**

#### **A) Distillation Framework**

```python
import torch.nn.functional as F

class LegalKnowledgeDistiller:
    """
    Distilla conoscenza da modello costoso (Opus) a modello economico (Haiku).
    
    Paper: "Distilling the Knowledge in a Neural Network" (Hinton et al., 2015)
    """
    
    def __init__(
        self, 
        teacher_model: str = "claude-3-opus",
        student_model: str = "claude-3-haiku",
        temperature: float = 3.0
    ):
        self.teacher = teacher_model
        self.student = student_model
        self.temperature = temperature
    
    async def distill_batch(
        self, 
        documents: list[Document],
        epochs: int = 5
    ):
        """
        Training loop per distillation.
        """
        for epoch in range(epochs):
            for doc in documents:
                # Teacher predictions (soft targets)
                teacher_logits = await self.get_teacher_logits(doc.text)
                teacher_soft = F.softmax(teacher_logits / self.temperature, dim=-1)
                
                # Student predictions
                student_logits = await self.get_student_logits(doc.text)
                student_soft = F.softmax(student_logits / self.temperature, dim=-1)
                
                # Distillation loss (KL divergence)
                distill_loss = F.kl_div(
                    student_soft.log(), 
                    teacher_soft, 
                    reduction='batchmean'
                ) * (self.temperature ** 2)
                
                # Ground truth loss (se annotato)
                if doc.label:
                    hard_loss = F.cross_entropy(student_logits, doc.label)
                    total_loss = 0.7 * distill_loss + 0.3 * hard_loss
                else:
                    total_loss = distill_loss
                
                # Backprop (pseudo-code per LLM fine-tuning)
                await self.update_student(total_loss)
    
    async def get_teacher_logits(self, text: str) -> torch.Tensor:
        """
        Chiamata a Opus con log_probs per ogni classe.
        """
        response = await litellm.completion(
            model=self.teacher,
            messages=[{
                "role": "user",
                "content": f"Classify legal concept: {text}\nReturn logits for each concept."
            }],
            logprobs=True,
            top_logprobs=50
        )
        
        # Estrai logits dalle logprobs API
        logits = self._extract_logits(response)
        return torch.tensor(logits)
```

#### **B) Contrastive Learning per Hard Negatives**

```python
from sentence_transformers import SentenceTransformer, losses, InputExample

class ContrastiveLegalLearner:
    """
    Migliora embeddings con contrastive learning su hard negatives.
    
    Paper: "SimCLR" (Chen et al., 2020)
    """
    
    def __init__(self, base_model: str = "multilingual-e5-large"):
        self.model = SentenceTransformer(base_model)
    
    def mine_hard_negatives(
        self, 
        anchor_doc: Document,
        corpus: list[Document],
        k: int = 5
    ) -> list[Document]:
        """
        Trova documenti semanticamente simili ma con label diverso.
        """
        anchor_emb = self.model.encode(anchor_doc.text)
        
        # Calcola similarità con corpus
        corpus_embs = self.model.encode([d.text for d in corpus])
        similarities = cosine_similarity([anchor_emb], corpus_embs)[0]
        
        # Hard negatives: alta similarità ma label diverso
        hard_negs = []
        for idx, sim in enumerate(similarities):
            if (corpus[idx].label != anchor_doc.label and 
                0.6 < sim < 0.85):  # Sweet spot
                hard_negs.append((sim, corpus[idx]))
        
        # Top-K più difficili
        hard_negs.sort(reverse=True)
        return [doc for _, doc in hard_negs[:k]]
    
    def create_training_examples(
        self, 
        annotated_docs: list[Document]
    ) -> list[InputExample]:
        """
        Crea triplets (anchor, positive, negative).
        """
        examples = []
        
        for anchor in annotated_docs:
            # Positive: stesso label
            positives = [d for d in annotated_docs if d.label == anchor.label and d.id != anchor.id]
            
            # Hard negatives
            hard_negs = self.mine_hard_negatives(anchor, annotated_docs)
            
            for pos in positives[:2]:  # Max 2 positives per anchor
                for neg in hard_negs[:3]:  # Max 3 negatives
                    examples.append(InputExample(
                        texts=[anchor.text, pos.text, neg.text],
                        label=1.0  # Triplet loss
                    ))
        
        return examples
    
    def train(self, examples: list[InputExample], epochs: int = 3):
        """
        Fine-tune con contrastive loss.
        """
        train_dataloader = DataLoader(examples, shuffle=True, batch_size=16)
        
        # Triplet loss con margin
        train_loss = losses.TripletLoss(
            model=self.model,
            distance_metric=losses.TripletDistanceMetric.COSINE,
            triplet_margin=0.5
        )
        
        self.model.fit(
            train_objectives=[(train_dataloader, train_loss)],
            epochs=epochs,
            warmup_steps=100,
            evaluation_steps=500
        )
```

---

### **5. Continual Learning (Evita Catastrophic Forgetting)**

```python
from torch import nn
import copy

class ElasticWeightConsolidation:
    """
    EWC: penalizza cambiamenti ai pesi importanti per task vecchi.
    
    Paper: "Overcoming catastrophic forgetting" (Kirkpatrick et al., 2017)
    """
    
    def __init__(self, model: nn.Module, lambda_ewc: float = 0.4):
        self.model = model
        self.lambda_ewc = lambda_ewc
        
        # Salva pesi iniziali
        self.old_params = {
            name: param.clone().detach()
            for name, param in model.named_parameters()
        }
        
        # Fisher information matrix (importanza pesi)
        self.fisher = self._compute_fisher()
    
    def _compute_fisher(self) -> dict:
        """
        Stima Fisher Information via gradients.
        """
        fisher = {}
        
        # Calcola gradients su dataset di validazione
        for name, param in self.model.named_parameters():
            fisher[name] = param.grad.data.clone().pow(2)
        
        return fisher
    
    def penalty(self) -> torch.Tensor:
        """
        EWC penalty term da aggiungere alla loss.
        """
        loss = 0
        
        for name, param in self.model.named_parameters():
            old_param = self.old_params[name]
            fisher_diag = self.fisher[name]
            
            # Penalizza cambiamenti proporzionalmente a Fisher
            loss += (fisher_diag * (param - old_param).pow(2)).sum()
        
        return self.lambda_ewc / 2 * loss

# Usage in training loop
ewc = ElasticWeightConsolidation(model, lambda_ewc=0.4)

for batch in new_data:
    # Standard loss
    task_loss = model.compute_loss(batch)
    
    # EWC penalty
    ewc_loss = ewc.penalty()
    
    # Total loss
    total_loss = task_loss + ewc_loss
    total_loss.backward()
    optimizer.step()
```

---

### **6. Few-Shot Prompt Optimization con DSPy**

```python
import dspy

class LegalConceptClassifier(dspy.Module):
    """
    DSPy signature per classificazione concetti legali.
    """
    
    def __init__(self):
        super().__init__()
        
        # Definisci signature
        self.classify = dspy.ChainOfThought("text -> concept, reasoning")
    
    def forward(self, text: str):
        prediction = self.classify(text=text)
        return prediction.concept, prediction.reasoning

# Auto-optimization con MIPROv2
from dspy.teleprompt import MIPROv2

teleprompter = MIPROv2(
    metric=legal_accuracy_metric,
    num_candidates=10,
    init_temperature=1.0
)

# Ottimizza su training set
optimized_classifier = teleprompter.compile(
    LegalConceptClassifier(),
    trainset=annotated_examples[:100],
    valset=annotated_examples[100:150]
)

# Deploy prompt ottimizzato
optimized_classifier("Art. 2043 c.c. - responsabilità extracontrattuale")
```

---

## 📊 **Monitoring & Evaluation Framework**

### **A) Concept Drift Detection**

```python
from scipy.stats import ks_2samp
from alibi_detect.cd import KSDrift

class ConceptDriftMonitor:
    """
    Monitora drift nella distribuzione dei concetti estratti.
    """
    
    def __init__(self, reference_embeddings: np.ndarray):
        self.detector = KSDrift(
            reference_embeddings,
            p_val=0.05,
            correction='bonferroni'
        )
    
    async def check_drift(self, new_embeddings: np.ndarray) -> dict:
        """
        Testa se nuovi documenti hanno distribuzione diversa.
        """
        result = self.detector.predict(new_embeddings)
        
        if result['data']['is_drift']:
            # Trigger retraining
            await self.trigger_active_learning_cycle()
            
            return {
                "drift_detected": True,
                "p_value": result['data']['p_val'],
                "distance": result['data']['distance'],
                "action": "retrain_scheduled"
            }
        
        return {"drift_detected": False}
```

### **B) Learning Curve Analytics**

```python
import matplotlib.pyplot as plt
from sklearn.model_selection import learning_curve

class LearningCurveAnalyzer:
    """
    Analizza efficacia dell'active learning.
    """
    
    def plot_learning_curve(
        self, 
        model,
        X: np.ndarray,
        y: np.ndarray,
        title: str = "Legal Concept Learning Curve"
    ):
        """
        Plotta accuracy vs numero di esempi annotati.
        """
        train_sizes, train_scores, val_scores = learning_curve(
            model, X, y,
            train_sizes=np.linspace(0.1, 1.0, 10),
            cv=5,
            scoring='f1_macro',
            n_jobs=-1
        )
        
        plt.figure(figsize=(10, 6))
        plt.plot(train_sizes, train_scores.mean(axis=1), label='Train')
        plt.plot(train_sizes, val_scores.mean(axis=1), label='Validation')
        plt.xlabel('Number of Annotated Examples')
        plt.ylabel('F1 Score')
        plt.title(title)
        plt.legend()
        plt.grid(True)
        plt.savefig('learning_curve.png')
        
        # Stima samples necessari per target performance
        target_f1 = 0.95
        samples_needed = self._extrapolate_samples(
            train_sizes, val_scores, target_f1
        )
        
        return {
            "current_f1": val_scores[-1].mean(),
            "samples_annotated": len(y),
            "estimated_samples_for_95": samples_needed
        }
```

---

## 🎯 **Implementation Roadmap**

### **Week 1-2: Foundation**
- [ ] Setup Bayesian classifier con MC Dropout
- [ ] Implement ensemble voting (3 models)
- [ ] Build semantic uncertainty detector
- [ ] Create annotation database schema

### **Week 3-4: Active Sampling**
- [ ] Implement hybrid sampling strategy
- [ ] Build citation graph PageRank
- [ ] Deploy annotation UI (Streamlit)
- [ ] Setup multi-annotator workflow

### **Week 5-6: Distillation**
- [ ] Train teacher model (Opus) su gold set
- [ ] Implement KD pipeline (Opus → Sonnet)
- [ ] Setup contrastive learning
- [ ] Benchmark student vs teacher

### **Week 7-8: Continual Learning**
- [ ] Implement EWC for continual training
- [ ] Setup drift detection monitoring
- [ ] Build automated retraining pipeline
- [ ] Deploy canary testing

### **Week 9-10: Production**
- [ ] A/B test optimized prompts (DSPy)
- [ ] Performance benchmarking
- [ ] Cost analysis (annotation vs automation)
- [ ] Go-live with monitoring

---

## 💰 **Cost-Benefit Analysis**

| Component | Setup Cost | Monthly Cost | ROI |
|-----------|------------|--------------|-----|
| **Annotation Platform** | €500 (dev) | €200 (hosting) | -€700 |
| **Human Annotators** (3 lawyers, 10h/week) | €0 | €1,200 | -€1,200 |
| **Teacher Model Calls** (Opus, 50K tokens/week) | €0 | €150 | -€150 |
| **Automated Improvements** | €0 | €0 | +€3,000
* |
| **Net** | **€500** | **€1,550** | **+€950/month** |

*Stima basata su:
- 20% riduzione errori → -5h revisione manuale/settimana → €500/mese
- 30% più accuratezza → +10 clienti soddisfatti → €2,500/mese valore brand

**Breakeven Point**: Mese 4 (dopo €6,200 investimento iniziale)

---

## 🚀 **Success Metrics**

| Metric | Baseline | Target (3 mesi) | Target (6 mesi) |
|--------|----------|-----------------|-----------------|
| **F1 Score (concetti rari)** | 0.72 | 0.85 | 0.92 |
| **Annotation Hours/Week** | 20h | 10h | 5h |
| **Model Confidence (avg)** | 0.68 | 0.82 | 0.90 |
| **Concepts Covered** | 80/150 | 120/150 | 145/150 |
| **Inter-Annotator Agreement** | 0.65 | 0.80 | 0.88 |
| **Cost per Annotation** | €15 | €8 | €3 |

---

Questa architettura crea un **flywheel effect**: più documenti annotati → modello migliore → meno incertezza → meno annotazioni necessarie → costi decrescenti. È un sistema che **impara a imparare** in modo sempre più efficiente. 🧠✨