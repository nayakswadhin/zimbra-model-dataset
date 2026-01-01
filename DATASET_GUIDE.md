# Dataset Usage Guide

## 📚 Quick Start Guide

This guide provides practical examples and best practices for using the Zimbra Java Vulnerability Dataset.

---

## 🎯 Table of Contents

1. [Loading the Dataset](#loading-the-dataset)
2. [Data Exploration](#data-exploration)
3. [Training Models](#training-models)
4. [Evaluation](#evaluation)
5. [Sample Analysis](#sample-analysis)
6. [Best Practices](#best-practices)
7. [Troubleshooting](#troubleshooting)

---

## Loading the Dataset

### Python (Basic)

```python
import json

# Load training data
with open('data/authentic/train.json', 'r', encoding='utf-8') as f:
    train_data = json.load(f)

# Load validation data
with open('data/authentic/val.json', 'r', encoding='utf-8') as f:
    val_data = json.load(f)

# Load test data
with open('data/authentic/test.json', 'r', encoding='utf-8') as f:
    test_data = json.load(f)

print(f"Training samples: {len(train_data)}")
print(f"Validation samples: {len(val_data)}")
print(f"Test samples: {len(test_data)}")
```

### Using Pandas

```python
import pandas as pd

# Load as DataFrame
train_df = pd.read_json('data/authentic/train.json')
val_df = pd.read_json('data/authentic/val.json')
test_df = pd.read_json('data/authentic/test.json')

# Display basic info
print(train_df.info())
print(train_df.head())
```

### Using HuggingFace Datasets

```python
from datasets import Dataset, DatasetDict

# Load JSON files
with open('data/authentic/train.json', 'r') as f:
    train_data = json.load(f)
with open('data/authentic/val.json', 'r') as f:
    val_data = json.load(f)
with open('data/authentic/test.json', 'r') as f:
    test_data = json.load(f)

# Create HuggingFace Dataset
dataset = DatasetDict({
    'train': Dataset.from_list(train_data),
    'validation': Dataset.from_list(val_data),
    'test': Dataset.from_list(test_data)
})

print(dataset)
```

---

## Data Exploration

### Basic Statistics

```python
import json
from collections import Counter

with open('data/authentic/train.json', 'r') as f:
    data = json.load(f)

# Vulnerability type distribution
vuln_types = [sample['vulnerability_type'] for sample in data]
print("Vulnerability Distribution:")
for vuln, count in Counter(vuln_types).items():
    print(f"  {vuln}: {count}")

# Confidence score distribution
confidence_scores = [sample['confidence_score'] for sample in data]
print(f"\nConfidence Scores:")
print(f"  Average: {sum(confidence_scores) / len(confidence_scores):.2f}")
print(f"  Min: {min(confidence_scores)}")
print(f"  Max: {max(confidence_scores)}")

# Repository distribution
repos = [sample['repo'] for sample in data]
print("\nRepository Distribution:")
for repo, count in Counter(repos).most_common():
    print(f"  {repo}: {count}")

# Code length statistics
code_lengths = [len(sample['vulnerable_code']) for sample in data]
print(f"\nCode Length Statistics:")
print(f"  Average: {sum(code_lengths) / len(code_lengths):.0f} characters")
print(f"  Min: {min(code_lengths)}")
print(f"  Max: {max(code_lengths)}")
```

### Filtering Samples

```python
# Filter by vulnerability type
sql_injections = [s for s in data if s['vulnerability_type'] == 'SQL Injection']
print(f"SQL Injection samples: {len(sql_injections)}")

# Filter by confidence score
high_confidence = [s for s in data if s['confidence_score'] >= 0.8]
print(f"High-confidence samples (≥0.8): {len(high_confidence)}")

# Filter by repository
mailbox_samples = [s for s in data if s['repo'] == 'zm-mailbox']
print(f"zm-mailbox samples: {len(mailbox_samples)}")

# Filter by code length (e.g., short samples)
short_samples = [s for s in data if len(s['vulnerable_code']) < 1000]
print(f"Short code samples (<1000 chars): {len(short_samples)}")
```

### Sample Inspection

```python
# Get a random sample
import random
sample = random.choice(data)

print("=" * 70)
print(f"Sample #{sample['serial_no']}")
print("=" * 70)
print(f"Vulnerability Type: {sample['vulnerability_type']}")
print(f"Repository: {sample['repo']}")
print(f"File: {sample['original_file']}")
print(f"Confidence: {sample['confidence_score']}")
print(f"Commit: {sample['commit']}")
print(f"\nCommit Message:")
print(sample['commit_msg'])
print(f"\nVerification URL:")
print(f"https://github.com/Zimbra/{sample['repo']}/commit/{sample['commit']}")
print(f"\nVulnerable Code (first 500 chars):")
print(sample['vulnerable_code'][:500] + "...")
```

---

## Training Models

### Example 1: Fine-tuning CodeBERT

```python
from transformers import (
    AutoTokenizer, 
    AutoModelForSequenceClassification,
    TrainingArguments,
    Trainer
)
from datasets import Dataset
import torch

# Load dataset
with open('data/authentic/train.json', 'r') as f:
    train_data = json.load(f)
with open('data/authentic/val.json', 'r') as f:
    val_data = json.load(f)

# Create label mapping
label_map = {
    "SQL Injection": 0,
    "Cross-Site Scripting (XSS)": 1,
    "Command Injection": 2,
    "Path Traversal": 3,
    "Insecure Deserialization": 4
}

# Prepare datasets
train_dataset = Dataset.from_dict({
    'code': [s['vulnerable_code'] for s in train_data],
    'label': [label_map[s['vulnerability_type']] for s in train_data]
})

val_dataset = Dataset.from_dict({
    'code': [s['vulnerable_code'] for s in val_data],
    'label': [label_map[s['vulnerability_type']] for s in val_data]
})

# Load tokenizer and model
tokenizer = AutoTokenizer.from_pretrained("microsoft/codebert-base")
model = AutoModelForSequenceClassification.from_pretrained(
    "microsoft/codebert-base",
    num_labels=5
)

# Tokenize function
def tokenize_function(examples):
    return tokenizer(
        examples['code'],
        padding='max_length',
        truncation=True,
        max_length=512
    )

# Tokenize datasets
train_dataset = train_dataset.map(tokenize_function, batched=True)
val_dataset = val_dataset.map(tokenize_function, batched=True)

# Training arguments
training_args = TrainingArguments(
    output_dir='./results',
    num_train_epochs=3,
    per_device_train_batch_size=8,
    per_device_eval_batch_size=8,
    warmup_steps=500,
    weight_decay=0.01,
    logging_dir='./logs',
    logging_steps=10,
    evaluation_strategy="epoch",
    save_strategy="epoch",
    load_best_model_at_end=True,
)

# Create Trainer
trainer = Trainer(
    model=model,
    args=training_args,
    train_dataset=train_dataset,
    eval_dataset=val_dataset,
)

# Train
trainer.train()

# Save model
trainer.save_model('./finetuned-vulnerability-detector')
```

### Example 2: Custom PyTorch Model

```python
import torch
import torch.nn as nn
from torch.utils.data import Dataset, DataLoader
from transformers import AutoTokenizer

class VulnerabilityDataset(Dataset):
    def __init__(self, json_file, tokenizer, max_length=512):
        with open(json_file, 'r') as f:
            self.data = json.load(f)
        self.tokenizer = tokenizer
        self.max_length = max_length
        self.label_map = {
            "SQL Injection": 0,
            "Cross-Site Scripting (XSS)": 1,
            "Command Injection": 2,
            "Path Traversal": 3,
            "Insecure Deserialization": 4
        }
    
    def __len__(self):
        return len(self.data)
    
    def __getitem__(self, idx):
        sample = self.data[idx]
        
        # Tokenize code
        encoded = self.tokenizer(
            sample['vulnerable_code'],
            padding='max_length',
            truncation=True,
            max_length=self.max_length,
            return_tensors='pt'
        )
        
        return {
            'input_ids': encoded['input_ids'].squeeze(),
            'attention_mask': encoded['attention_mask'].squeeze(),
            'label': torch.tensor(self.label_map[sample['vulnerability_type']])
        }

# Initialize
tokenizer = AutoTokenizer.from_pretrained("microsoft/codebert-base")

# Create datasets
train_dataset = VulnerabilityDataset('data/authentic/train.json', tokenizer)
val_dataset = VulnerabilityDataset('data/authentic/val.json', tokenizer)

# Create dataloaders
train_loader = DataLoader(train_dataset, batch_size=8, shuffle=True)
val_loader = DataLoader(val_dataset, batch_size=8)

# Your training loop here...
```

---

## Evaluation

### Computing Metrics

```python
from sklearn.metrics import (
    accuracy_score,
    precision_recall_fscore_support,
    classification_report,
    confusion_matrix
)
import numpy as np

def evaluate_model(model, tokenizer, test_file):
    """Evaluate model on test set"""
    
    # Load test data
    with open(test_file, 'r') as f:
        test_data = json.load(f)
    
    label_map = {
        "SQL Injection": 0,
        "Cross-Site Scripting (XSS)": 1,
        "Command Injection": 2,
        "Path Traversal": 3,
        "Insecure Deserialization": 4
    }
    
    # Prepare inputs
    codes = [s['vulnerable_code'] for s in test_data]
    true_labels = [label_map[s['vulnerability_type']] for s in test_data]
    
    # Get predictions
    predictions = []
    for code in codes:
        inputs = tokenizer(code, return_tensors='pt', truncation=True, max_length=512)
        outputs = model(**inputs)
        pred = torch.argmax(outputs.logits, dim=1).item()
        predictions.append(pred)
    
    # Compute metrics
    accuracy = accuracy_score(true_labels, predictions)
    precision, recall, f1, _ = precision_recall_fscore_support(
        true_labels, predictions, average='weighted'
    )
    
    print(f"Accuracy: {accuracy:.4f}")
    print(f"Precision: {precision:.4f}")
    print(f"Recall: {recall:.4f}")
    print(f"F1-Score: {f1:.4f}")
    
    # Detailed classification report
    label_names = list(label_map.keys())
    print("\nClassification Report:")
    print(classification_report(true_labels, predictions, target_names=label_names))
    
    # Confusion matrix
    cm = confusion_matrix(true_labels, predictions)
    print("\nConfusion Matrix:")
    print(cm)
    
    return {
        'accuracy': accuracy,
        'precision': precision,
        'recall': recall,
        'f1': f1,
        'confusion_matrix': cm
    }

# Usage
results = evaluate_model(model, tokenizer, 'data/authentic/test.json')
```

### Per-Class Analysis

```python
def analyze_per_class_performance(y_true, y_pred, label_names):
    """Analyze performance for each vulnerability class"""
    
    precision, recall, f1, support = precision_recall_fscore_support(
        y_true, y_pred, average=None
    )
    
    print("Per-Class Performance:")
    print("-" * 70)
    for i, label in enumerate(label_names):
        print(f"{label}:")
        print(f"  Precision: {precision[i]:.4f}")
        print(f"  Recall:    {recall[i]:.4f}")
        print(f"  F1-Score:  {f1[i]:.4f}")
        print(f"  Support:   {support[i]}")
        print()
```

---

## Sample Analysis

### Verifying Sample Authenticity

```python
def verify_sample(sample):
    """Generate verification information for a sample"""
    
    repo = sample['repo']
    commit = sample['commit']
    
    print(f"Sample Verification Report")
    print("=" * 70)
    print(f"Serial No: {sample['serial_no']}")
    print(f"Vulnerability Type: {sample['vulnerability_type']}")
    print(f"Confidence Score: {sample['confidence_score']}")
    print()
    print(f"Source:")
    print(f"  Repository: {repo}")
    print(f"  File: {sample['original_file']}")
    print(f"  Commit: {commit}")
    print()
    print(f"Verification URL:")
    print(f"  https://github.com/Zimbra/{repo}/commit/{commit}")
    print()
    print(f"Commit Message:")
    print(f"  {sample['commit_msg']}")
    print()
    print(f"To verify:")
    print(f"  1. Visit the URL above")
    print(f"  2. Check the 'Files changed' tab")
    print(f"  3. Look for {sample['original_file']}")
    print(f"  4. Verify the vulnerable code exists in the 'before' state")
    print("=" * 70)

# Verify a specific sample
with open('data/authentic/train.json', 'r') as f:
    data = json.load(f)

verify_sample(data[0])
```

### Finding Similar Samples

```python
from difflib import SequenceMatcher

def find_similar_samples(target_sample, dataset, top_k=5):
    """Find samples similar to the target sample"""
    
    target_code = target_sample['vulnerable_code']
    similarities = []
    
    for sample in dataset:
        if sample['serial_no'] == target_sample['serial_no']:
            continue
        
        # Calculate similarity
        similarity = SequenceMatcher(
            None, 
            target_code, 
            sample['vulnerable_code']
        ).ratio()
        
        similarities.append((sample, similarity))
    
    # Sort by similarity
    similarities.sort(key=lambda x: x[1], reverse=True)
    
    print(f"Top {top_k} similar samples to #{target_sample['serial_no']}:")
    print("-" * 70)
    for i, (sample, sim) in enumerate(similarities[:top_k], 1):
        print(f"{i}. Sample #{sample['serial_no']}")
        print(f"   Similarity: {sim:.2%}")
        print(f"   Type: {sample['vulnerability_type']}")
        print(f"   Repo: {sample['repo']}")
        print()

# Usage
target = data[0]
find_similar_samples(target, data)
```

---

## Best Practices

### 1. Data Loading

✅ **DO:**
- Use UTF-8 encoding when loading JSON files
- Load validation/test data separately from training
- Implement proper error handling for missing files

❌ **DON'T:**
- Mix training and test data
- Modify test data during training
- Use test data for hyperparameter tuning

### 2. Preprocessing

✅ **DO:**
- Truncate long code samples to model's max length
- Use proper tokenization for code (e.g., CodeBERT tokenizer)
- Preserve code structure during tokenization

❌ **DON'T:**
- Remove all whitespace/formatting
- Use word-level tokenizers for code
- Lose syntactic structure

### 3. Training

✅ **DO:**
- Use the validation set for early stopping
- Save best model based on validation metrics
- Track per-class performance
- Use appropriate learning rates for fine-tuning

❌ **DON'T:**
- Train without validation monitoring
- Use test data during training
- Ignore class imbalance (if using augmented data)

### 4. Evaluation

✅ **DO:**
- Report multiple metrics (accuracy, precision, recall, F1)
- Analyze per-class performance
- Use the test set only once for final evaluation
- Include confidence intervals if possible

❌ **DON'T:**
- Report only accuracy
- Evaluate multiple times on test set
- Cherry-pick best results

### 5. Reproducibility

✅ **DO:**
- Set random seeds
- Document hyperparameters
- Save model checkpoints
- Record training logs

❌ **DON'T:**
- Skip documentation
- Use undocumented hyperparameters
- Delete intermediate results

---

## Troubleshooting

### Issue: JSON Decode Error

```python
# Problem: File encoding issues
# Solution: Specify UTF-8 encoding
with open('data/authentic/train.json', 'r', encoding='utf-8') as f:
    data = json.load(f)
```

### Issue: Out of Memory

```python
# Problem: Code samples too long
# Solution: Truncate or use sliding window

def truncate_code(code, max_length=512, tokenizer=None):
    if tokenizer:
        tokens = tokenizer.encode(code, truncation=True, max_length=max_length)
        return tokenizer.decode(tokens)
    else:
        return code[:max_length]
```

### Issue: Class Imbalance (Augmented Data)

```python
# Solution: Use class weights
from sklearn.utils.class_weight import compute_class_weight

# Compute class weights
class_weights = compute_class_weight(
    'balanced',
    classes=np.unique(labels),
    y=labels
)

# Use in PyTorch
criterion = nn.CrossEntropyLoss(weight=torch.FloatTensor(class_weights))
```

### Issue: Slow Training

```python
# Solution: Use DataLoader with multiple workers
train_loader = DataLoader(
    train_dataset,
    batch_size=16,
    shuffle=True,
    num_workers=4,  # Parallel data loading
    pin_memory=True  # Faster GPU transfer
)
```

---

## Additional Resources

### Code Examples Repository
- Full training scripts
- Evaluation notebooks
- Data analysis tools
- Visualization code

### Recommended Tools
- **PyDriller**: Git repository mining
- **Transformers**: Pre-trained models
- **scikit-learn**: Evaluation metrics
- **pandas**: Data manipulation

### Further Reading
- OWASP Vulnerability Classification
- CodeBERT Paper
- Vulnerability Detection Survey Papers

---

**Document Version**: 1.0  
**Last Updated**: January 2026
