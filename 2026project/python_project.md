# 監督式學習－標題黨偵測分類（Clickbait Classification）Python 實作手冊

> **對應規劃文件**：`監督式學習分類專案規劃書.md`  
> **資料集路徑**：`2026project/clickbait_data.csv`  
> **Python 版本**：Python 3.12+  

本文件依照專案規劃書的 **環境建置** 與 **Phase 1 ～ Phase 5** 各階段與步驟，提供對應且可直接執行的 Python 程式碼。

---

## Phase 0：環境建置與專案目錄初始化（對應規劃書第 7 節）

### 步驟 0.1：建立專案資料夾架構與虛擬環境

在終端機（PowerShell / Bash）中建立目錄並使用 `uv` 安裝相依套件：

```bash
# 建立專案目錄結構
mkdir -p 2026project/notebooks 2026project/src 2026project/models 2026project/reports
cd 2026project

# 使用 uv 初始化虛擬環境並安裝套件
uv init --python 3.12
uv add pandas numpy scikit-learn matplotlib seaborn lightgbm joblib
```

建立 `src/__init__.py`（空檔案或模組匯出）：

```python
# src/__init__.py
"""Clickbait Classification 專案核心模組。"""
```

---

## Phase 1：探索性資料分析（EDA）與資料切分（對應規劃書第 3、4.1、8 節）

### 步驟 1.1：資料集現況盤點與品質檢核

對應規劃書 **3.1 資料集現況盤點**：檢查總筆數、欄位名稱、類別分布、缺失值、重複標題及字數統計。

```python
# notebooks/01_eda_clickbait.py (或於 01_eda_clickbait.ipynb 中執行)
from pathlib import Path
import pandas as pd

DATA_PATH = Path("clickbait_data.csv")
df = pd.read_csv(DATA_PATH)

print("=== 1. 資料集基本資訊 ===")
print(f"總樣本數: {len(df):,}")
print(f"欄位名稱: {df.columns.tolist()}")
print("\n前 5 筆資料:")
print(df.head())

print("\n=== 2. 類別分布 (clickbait) ===")
counts = df["clickbait"].value_counts()
ratios = df["clickbait"].value_counts(normalize=True) * 100
for label in counts.index:
    print(f"類別 {label}: {counts[label]:,} 筆 ({ratios[label]:.3f}%)")

print("\n=== 3. 缺失值與重複值檢核 ===")
print("缺失值統計:\n", df.isnull().sum())
print(f"重複標題數: {df['headline'].duplicated().sum()}")

print("\n=== 4. 單篇字數統計 ===")
word_counts = df["headline"].astype(str).str.split().str.len()
print(word_counts.describe()[["min", "mean", "50%", "max"]])
```

### 步驟 1.2：探索性資料分析（EDA）與視覺化

對應規劃書 **4.1 EDA 重點**：比較兩類別標題長度分布、標點符號（`?`、`!`）、數字開頭、人稱代詞（`you` / `your`）頻率，以及繪製高頻 Uni-gram / Bi-gram 關鍵字排行。

```python
from pathlib import Path
import re
import pandas as pd
import matplotlib.pyplot as plt
import seaborn as sns
from sklearn.feature_extraction.text import CountVectorizer

REPORTS_DIR = Path("reports")
REPORTS_DIR.mkdir(parents=True, exist_ok=True)

df = pd.read_csv("clickbait_data.csv")
df["char_len"] = df["headline"].str.len()
df["word_len"] = df["headline"].str.split().str.len()
df["has_question"] = df["headline"].str.contains(r"\?", regex=True)
df["has_exclamation"] = df["headline"].str.contains(r"!", regex=True)
df["starts_with_number"] = df["headline"].str.match(r"^\d+")
df["has_you_your"] = df["headline"].str.contains(r"\b(you|your|you're)\b", case=False, regex=True)

# 1. 句型與標點特徵比例比較
feature_cols = ["has_question", "has_exclamation", "starts_with_number", "has_you_your"]
print("=== 兩類別句型特徵比例 ===")
print(df.groupby("clickbait")[feature_cols].mean().round(4))

# 2. 繪製字數分布圖
plt.figure(figsize=(10, 5))
sns.histplot(data=df, x="word_len", hue="clickbait", bins=26, kde=True, stat="density", common_norm=False)
plt.title("Word Count Distribution: Clickbait (1) vs Non-Clickbait (0)")
plt.xlabel("Word Count")
plt.ylabel("Density")
plt.tight_layout()
plt.savefig(REPORTS_DIR / "eda_word_length_dist.png", dpi=150)
plt.close()

# 3. 高頻 Uni-gram / Bi-gram 排行比較
def plot_top_ngrams(texts: pd.Series, title: str, filename: str, ngram_range=(1, 2), top_n: int = 15) -> None:
    vec = CountVectorizer(ngram_range=ngram_range, stop_words="english", max_features=5000)
    X = vec.fit_transform(texts)
    freqs = X.sum(axis=0).A1
    vocab = vec.get_feature_names_out()
    top_df = pd.DataFrame({"ngram": vocab, "count": freqs}).sort_values("count", ascending=False).head(top_n)

    plt.figure(figsize=(9, 5))
    sns.barplot(data=top_df, x="count", y="ngram", palette="viridis", hue="ngram", legend=False)
    plt.title(title)
    plt.xlabel("Frequency")
    plt.ylabel("N-gram")
    plt.tight_layout()
    plt.savefig(REPORTS_DIR / filename, dpi=150)
    plt.close()

plot_top_ngrams(
    df.loc[df["clickbait"] == 1, "headline"],
    "Top 15 N-grams in Clickbait Headlines",
    "eda_top_ngrams_clickbait.png",
)
plot_top_ngrams(
    df.loc[df["clickbait"] == 0, "headline"],
    "Top 15 N-grams in Non-Clickbait Headlines",
    "eda_top_ngrams_non_clickbait.png",
)
```

### 步驟 1.3：分層資料切分（Train 70% / Val 15% / Test 15%）

對應規劃書 **3.2 資料切分策略** 與 `src/dataset.py`：使用 `train_test_split(..., stratify=y, random_state=42)` 維持各集合 50:50 類別比例，確保測試集完全隔離。

```python
# src/dataset.py
from pathlib import Path
from typing import Tuple
import pandas as pd
from sklearn.model_selection import train_test_split

RANDOM_STATE = 42


def load_and_split_data(
    csv_path: str | Path = "clickbait_data.csv",
    random_state: int = RANDOM_STATE,
) -> Tuple[pd.DataFrame, pd.DataFrame, pd.DataFrame]:
    """讀取資料集並依 70% / 15% / 15% 進行分層抽樣切分。"""
    df = pd.read_csv(csv_path)

    # 先切出 70% 訓練集 與 30% 暫存集 (Val + Test)
    train_df, temp_df = train_test_split(
        df,
        test_size=0.30,
        stratify=df["clickbait"],
        random_state=random_state,
    )

    # 再將 30% 暫存集平分為 15% 驗證集 與 15% 測試集
    val_df, test_df = train_test_split(
        temp_df,
        test_size=0.50,
        stratify=temp_df["clickbait"],
        random_state=random_state,
    )

    return (
        train_df.reset_index(drop=True),
        val_df.reset_index(drop=True),
        test_df.reset_index(drop=True),
    )


if __name__ == "__main__":
    train_df, val_df, test_df = load_and_split_data()
    for name, part in [("Train", train_df), ("Val", val_df), ("Test", test_df)]:
        pos_ratio = part["clickbait"].mean() * 100
        print(f"{name:5s}: {len(part):5,} 筆 | Clickbait 比例: {pos_ratio:.2f}%")
```

---

## Phase 2：文本前處理與 Baseline 建立（對應規劃書第 2.2、4.2、5.1、8 節）

### 步驟 2.1：文本清理函式

對應規劃書 **4.2 文本前處理**：正規化多餘空白與換行，保留數字與關鍵標點符號，避免標題黨語氣特徵流失。

```python
# src/preprocessing.py
import re
from typing import Iterable, List


def clean_headline(text: str) -> str:
    """清理單筆標題文字：正規化空白，保留數字與關鍵標點。"""
    text = str(text).strip()
    text = re.sub(r"\s+", " ", text)
    return text


def clean_headlines(texts: Iterable[str]) -> List[str]:
    """批次清理標題列表（可搭配 FunctionTransformer 放入 Pipeline）。"""
    return [clean_headline(t) for t in texts]
```

### 步驟 2.2：TF-IDF + Baseline 模型管線與交叉驗證

對應規劃書 **5.1 Baseline 模型**：以 `Pipeline` 串接 `TfidfVectorizer` 與 `MultinomialNB` / `LogisticRegression`（嚴防資料洩漏），於訓練集進行 5-Fold Stratified Cross-Validation，並在驗證集檢視是否達到 **F1-Score $\ge$ 0.90** 基準線。

```python
# src/baseline.py
from sklearn.pipeline import Pipeline
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.naive_bayes import MultinomialNB
from sklearn.linear_model import LogisticRegression
from sklearn.model_selection import StratifiedKFold, cross_validate
from sklearn.metrics import accuracy_score, f1_score, roc_auc_score

from src.dataset import load_and_split_data, RANDOM_STATE
from src.preprocessing import clean_headlines

train_df, val_df, _ = load_and_split_data()
X_train = clean_headlines(train_df["headline"])
y_train = train_df["clickbait"]
X_val = clean_headlines(val_df["headline"])
y_val = val_df["clickbait"]

baselines = {
    "MultinomialNB": MultinomialNB(alpha=1.0),
    "LogisticRegression": LogisticRegression(max_iter=1000, random_state=RANDOM_STATE),
}

cv = StratifiedKFold(n_splits=5, shuffle=True, random_state=RANDOM_STATE)

for name, clf in baselines.items():
    pipe = Pipeline([
        ("tfidf", TfidfVectorizer(ngram_range=(1, 2), max_features=20000, sublinear_tf=True)),
        ("clf", clf),
    ])

    # 5-Fold CV（僅在訓練集執行）
    cv_res = cross_validate(pipe, X_train, y_train, cv=cv, scoring=["accuracy", "f1", "roc_auc"], n_jobs=-1)
    print(f"\n[{name}] 5-Fold CV 平均表現:")
    print(f"  CV Accuracy: {cv_res['test_accuracy'].mean():.4f}")
    print(f"  CV F1-Score: {cv_res['test_f1'].mean():.4f}")
    print(f"  CV ROC-AUC : {cv_res['test_roc_auc'].mean():.4f}")

    # 擬合完整訓練集並於驗證集評估
    pipe.fit(X_train, y_train)
    val_pred = pipe.predict(X_val)
    val_prob = pipe.predict_proba(X_val)[:, 1]
    print(f"[{name}] Validation 表現:")
    print(f"  Val Accuracy: {accuracy_score(y_val, val_pred):.4f}")
    print(f"  Val F1-Score: {f1_score(y_val, val_pred):.4f}")
    print(f"  Val ROC-AUC : {roc_auc_score(y_val, val_prob):.4f}")
```

---

## Phase 3：模型迭代、特徵強化與超參數調校（對應規劃書第 4.2、5.1、5.2、8 節）

### 步驟 3.1：人工統計特徵抽取器（Custom Transformer）

對應規劃書 **4.2 人工統計特徵**：抽取字數、字元數、問號/驚嘆號、是否數字開頭、是否含 `you`/`your` 及大寫字母比例等特徵，並封裝為相容於 `scikit-learn` `Pipeline` 的轉換器。

```python
# src/features.py
import re
import numpy as np
import pandas as pd
from sklearn.base import BaseEstimator, TransformerMixin


class HeadlineStatsExtractor(BaseEstimator, TransformerMixin):
    """從英文標題抽取統計與句型特徵，輸出二維數值陣列。"""

    def fit(self, X, y=None):
        return self

    def transform(self, X) -> np.ndarray:
        series = pd.Series(X).astype(str)
        word_len = series.str.split().str.len().to_numpy()
        char_len = series.str.len().to_numpy()
        has_question = series.str.contains(r"\?", regex=True).astype(int).to_numpy()
        has_exclamation = series.str.contains(r"!", regex=True).astype(int).to_numpy()
        starts_with_num = series.str.match(r"^\s*\d+").astype(int).to_numpy()
        has_you_your = series.str.contains(r"\b(you|your|you're)\b", case=False, regex=True).astype(int).to_numpy()
        title_case_ratio = series.apply(
            lambda s: sum(1 for w in s.split() if w[:1].isupper()) / max(len(s.split()), 1)
        ).to_numpy()

        return np.column_stack([
            word_len,
            char_len,
            has_question,
            has_exclamation,
            starts_with_num,
            has_you_your,
            title_case_ratio,
        ])
```

### 步驟 3.2 & 3.3：多模型比較實驗、超參數調校與最佳模型儲存

對應規劃書 **5.1 候選模型清單**（`LogisticRegression`、`LinearSVC`、`SGDClassifier`、`LightGBM`）、**5.2 超參數調校**（`GridSearchCV`）與 `src/train.py`。

> 註：`LinearSVC` 預設無 `predict_proba`，透過 `CalibratedClassifierCV` 包裝後即可輸出校準機率。

```python
# src/train.py
from pathlib import Path
import joblib
import pandas as pd
from lightgbm import LGBMClassifier
from sklearn.calibration import CalibratedClassifierCV
from sklearn.decomposition import TruncatedSVD
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.linear_model import LogisticRegression, SGDClassifier
from sklearn.metrics import accuracy_score, f1_score, roc_auc_score, log_loss
from sklearn.model_selection import GridSearchCV, StratifiedKFold
from sklearn.pipeline import FeatureUnion, Pipeline
from sklearn.preprocessing import StandardScaler
from sklearn.svm import LinearSVC

from src.dataset import load_and_split_data, RANDOM_STATE
from src.features import HeadlineStatsExtractor
from src.preprocessing import clean_headlines

MODELS_DIR = Path("models")
REPORTS_DIR = Path("reports")
MODELS_DIR.mkdir(parents=True, exist_ok=True)
REPORTS_DIR.mkdir(parents=True, exist_ok=True)


def build_feature_union() -> FeatureUnion:
    """結合 TF-IDF 稀疏特徵與標準化後的人工統計特徵。"""
    return FeatureUnion([
        ("tfidf", TfidfVectorizer(ngram_range=(1, 2), max_features=20000, sublinear_tf=True)),
        ("stats", Pipeline([
            ("extract", HeadlineStatsExtractor()),
            ("scale", StandardScaler()),
        ])),
    ])


def main() -> None:
    train_df, val_df, _ = load_and_split_data()
    X_train = clean_headlines(train_df["headline"])
    y_train = train_df["clickbait"]
    X_val = clean_headlines(val_df["headline"])
    y_val = val_df["clickbait"]

    cv = StratifiedKFold(n_splits=5, shuffle=True, random_state=RANDOM_STATE)

    # 1. 多模型實驗比較
    candidates = {
        "LogisticRegression": Pipeline([
            ("features", build_feature_union()),
            ("clf", LogisticRegression(C=2.0, max_iter=1000, random_state=RANDOM_STATE)),
        ]),
        "LinearSVC_Calibrated": Pipeline([
            ("features", build_feature_union()),
            ("clf", CalibratedClassifierCV(LinearSVC(C=1.0, random_state=RANDOM_STATE), cv=3)),
        ]),
        "SGDClassifier_LogLoss": Pipeline([
            ("features", build_feature_union()),
            ("clf", SGDClassifier(loss="log_loss", alpha=1e-5, max_iter=1000, random_state=RANDOM_STATE)),
        ]),
        "LightGBM_SVD": Pipeline([
            ("features", FeatureUnion([
                ("tfidf_svd", Pipeline([
                    ("tfidf", TfidfVectorizer(ngram_range=(1, 2), max_features=20000, sublinear_tf=True)),
                    ("svd", TruncatedSVD(n_components=150, random_state=RANDOM_STATE)),
                ])),
                ("stats", HeadlineStatsExtractor()),
            ])),
            ("clf", LGBMClassifier(n_estimators=300, learning_rate=0.05, random_state=RANDOM_STATE, verbose=-1)),
        ]),
    }

    results = []
    for name, pipe in candidates.items():
        pipe.fit(X_train, y_train)
        val_pred = pipe.predict(X_val)
        val_prob = pipe.predict_proba(X_val)[:, 1]
        results.append({
            "Model": name,
            "Val_Accuracy": accuracy_score(y_val, val_pred),
            "Val_F1": f1_score(y_val, val_pred),
            "Val_ROC_AUC": roc_auc_score(y_val, val_prob),
            "Val_LogLoss": log_loss(y_val, val_prob),
        })

    results_df = pd.DataFrame(results).sort_values("Val_F1", ascending=False)
    print("=== 多模型驗證集比較表 ===")
    print(results_df.to_string(index=False))
    results_df.to_csv(REPORTS_DIR / "model_comparison.csv", index=False)

    # 2. 針對表現優異的線性管線進行超參數調校 (GridSearchCV)
    tune_pipe = Pipeline([
        ("features", build_feature_union()),
        ("clf", LogisticRegression(max_iter=1000, random_state=RANDOM_STATE)),
    ])

    param_grid = {
        "features__tfidf__ngram_range": [(1, 1), (1, 2)],
        "features__tfidf__min_df": [1, 2, 3],
        "features__tfidf__max_features": [20000, 30000],
        "clf__C": [0.5, 1.0, 2.0, 5.0],
    }

    grid = GridSearchCV(
        tune_pipe,
        param_grid=param_grid,
        cv=cv,
        scoring="f1",
        n_jobs=-1,
        verbose=1,
    )
    grid.fit(X_train, y_train)

    print(f"\n最佳超參數: {grid.best_params_}")
    print(f"最佳 CV F1-Score: {grid.best_score_:.4f}")

    best_model = grid.best_estimator_
    val_pred = best_model.predict(X_val)
    val_prob = best_model.predict_proba(X_val)[:, 1]
    print(
        f"調參後 Validation -> Accuracy: {accuracy_score(y_val, val_pred):.4f}, "
        f"F1: {f1_score(y_val, val_pred):.4f}, ROC-AUC: {roc_auc_score(y_val, val_prob):.4f}"
    )

    # 儲存最佳 Pipeline
    model_path = MODELS_DIR / "clickbait_pipeline.joblib"
    joblib.dump(best_model, model_path)
    print(f"最佳模型已儲存至: {model_path}")


if __name__ == "__main__":
    main()
```

---

## Phase 4：測試集驗證、可解釋性與錯誤分析（對應規劃書第 2.2、6、8 節）

### 步驟 4.1 ～ 4.4：測試集最終評估、圖表繪製、特徵詞權重解釋與誤判案例分析

對應規劃書 **6 模型評估指標** 與 `src/evaluate.py`：
1. 在完全隔離的測試集（Test Set, 4,800 筆）進行最終評估（檢核 Accuracy $\ge$ 0.95、F1 $\ge$ 0.95、ROC-AUC $\ge$ 0.98）與單筆推論延遲（< 20ms）。
2. 繪製混淆矩陣（Confusion Matrix）與 ROC 曲線至 `reports/`。
3. 輸出權重最高的前 20 個「標題黨特徵詞」與「正常新聞特徵詞」。
4. 匯出誤判樣本（FP 與 FN）至 `reports/error_analysis.csv` 供檢視。

```python
# src/evaluate.py
from pathlib import Path
import time
import joblib
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
import seaborn as sns
from sklearn.metrics import (
    accuracy_score,
    classification_report,
    confusion_matrix,
    f1_score,
    log_loss,
    precision_score,
    recall_score,
    roc_auc_score,
    roc_curve,
)

from src.dataset import load_and_split_data
from src.preprocessing import clean_headlines

MODEL_PATH = Path("models/clickbait_pipeline.joblib")
REPORTS_DIR = Path("reports")
REPORTS_DIR.mkdir(parents=True, exist_ok=True)


def evaluate_latency(model, sample_text: str, n_runs: int = 200) -> float:
    """量測單筆標題平均推論延遲（毫秒 ms）。"""
    start = time.perf_counter()
    for _ in range(n_runs):
        _ = model.predict_proba([sample_text])
    elapsed_ms = ((time.perf_counter() - start) / n_runs) * 1000
    return elapsed_ms


def explain_top_features(model, top_n: int = 20) -> None:
    """輸出線性模型權重最高的前 N 個標題黨與正常新聞特徵詞。"""
    tfidf = model.named_steps["features"].transformer_list[0][1]
    clf = model.named_steps["clf"]
    if not hasattr(clf, "coef_"):
        print("此分類器不具備線性 coef_ 屬性，略過權重詞彙解釋。")
        return

    tfidf_names = tfidf.get_feature_names_out()
    coefs = clf.coef_[0][: len(tfidf_names)]

    top_pos_idx = np.argsort(coefs)[-top_n:][::-1]
    top_neg_idx = np.argsort(coefs)[:top_n]

    print(f"\n=== Top {top_n} 標題黨特徵詞 (Clickbait = 1) ===")
    for idx in top_pos_idx:
        print(f"  {tfidf_names[idx]:25s} ({coefs[idx]:+.4f})")

    print(f"\n=== Top {top_n} 正常新聞特徵詞 (Non-Clickbait = 0) ===")
    for idx in top_neg_idx:
        print(f"  {tfidf_names[idx]:25s} ({coefs[idx]:+.4f})")


def main() -> None:
    _, _, test_df = load_and_split_data()
    X_test = clean_headlines(test_df["headline"])
    y_test = test_df["clickbait"]

    model = joblib.load(MODEL_PATH)

    # 1. 測試集指標計算
    y_pred = model.predict(X_test)
    y_prob = model.predict_proba(X_test)[:, 1]

    print("=== 測試集最終評估結果 (Test Set) ===")
    print(f"Accuracy : {accuracy_score(y_test, y_pred):.4f}")
    print(f"Precision: {precision_score(y_test, y_pred):.4f}")
    print(f"Recall   : {recall_score(y_test, y_pred):.4f}")
    print(f"F1-Score : {f1_score(y_test, y_pred):.4f}")
    print(f"ROC-AUC  : {roc_auc_score(y_test, y_prob):.4f}")
    print(f"Log Loss : {log_loss(y_test, y_prob):.4f}")
    print("\n分類報告:\n", classification_report(y_test, y_pred, digits=4))

    # 單筆延遲檢核 (< 20ms)
    latency_ms = evaluate_latency(model, X_test[0])
    print(f"單筆標題平均推論延遲: {latency_ms:.3f} ms")

    # 2. 繪製混淆矩陣
    cm = confusion_matrix(y_test, y_pred)
    plt.figure(figsize=(6, 5))
    sns.heatmap(cm, annot=True, fmt="d", cmap="Blues", xticklabels=["Non-Clickbait", "Clickbait"], yticklabels=["Non-Clickbait", "Clickbait"])
    plt.title("Test Set Confusion Matrix")
    plt.xlabel("Predicted")
    plt.ylabel("Actual")
    plt.tight_layout()
    plt.savefig(REPORTS_DIR / "test_confusion_matrix.png", dpi=150)
    plt.close()

    # 3. 繪製 ROC 曲線
    fpr, tpr, _ = roc_curve(y_test, y_prob)
    auc_val = roc_auc_score(y_test, y_prob)
    plt.figure(figsize=(6, 5))
    plt.plot(fpr, tpr, label=f"ROC Curve (AUC = {auc_val:.4f})")
    plt.plot([0, 1], [0, 1], "k--")
    plt.xlabel("False Positive Rate")
    plt.ylabel("True Positive Rate")
    plt.title("Test Set ROC Curve")
    plt.legend(loc="lower right")
    plt.tight_layout()
    plt.savefig(REPORTS_DIR / "test_roc_curve.png", dpi=150)
    plt.close()

    # 4. 模型可解釋性
    explain_top_features(model, top_n=20)

    # 5. 誤判案例分析 (Error Analysis)
    analysis_df = test_df.copy()
    analysis_df["pred"] = y_pred
    analysis_df["prob_clickbait"] = y_prob.round(4)
    errors_df = analysis_df[analysis_df["clickbait"] != analysis_df["pred"]]
    errors_df.to_csv(REPORTS_DIR / "error_analysis.csv", index=False)

    print(f"\n誤判總筆數: {len(errors_df)} / {len(test_df)}")
    print("誤判範例前 10 筆:")
    print(errors_df[["headline", "clickbait", "pred", "prob_clickbait"]].head(10).to_string(index=False))


if __name__ == "__main__":
    main()
```

---

## Phase 5：模型封裝與推論工具（對應規劃書第 2.1、7.2、8、10 節）

### 步驟 5.1 & 5.2：建立單筆／批次標題推論介面與 CLI 工具

對應規劃書 **2.1 推論方式**（支援單筆／批次標題推論，輸出類別標籤與 `predict_proba`）與 `src/predict.py`。

```python
# src/predict.py
import argparse
from pathlib import Path
from typing import Dict, List, Union
import joblib

from src.preprocessing import clean_headlines

DEFAULT_MODEL_PATH = Path("models/clickbait_pipeline.joblib")


class ClickbaitDetector:
    """載入已封裝的 Pipeline 模型並提供單筆與批次標題推論。"""

    def __init__(self, model_path: str | Path = DEFAULT_MODEL_PATH) -> None:
        self.model = joblib.load(model_path)

    def predict(self, headlines: Union[str, List[str]], threshold: float = 0.5) -> List[Dict]:
        is_single = isinstance(headlines, str)
        texts = [headlines] if is_single else list(headlines)
        cleaned = clean_headlines(texts)

        probs = self.model.predict_proba(cleaned)[:, 1]
        results = []
        for raw_text, prob in zip(texts, probs):
            label = int(prob >= threshold)
            results.append({
                "headline": raw_text,
                "label": label,
                "label_name": "Clickbait" if label == 1 else "Non-Clickbait",
                "clickbait_probability": round(float(prob), 4),
            })
        return results


def main() -> None:
    parser = argparse.ArgumentParser(description="英文標題黨偵測推論工具")
    parser.add_argument("headlines", nargs="+", help="輸入一個或多個英文標題")
    parser.add_argument("--model", default=str(DEFAULT_MODEL_PATH), help="模型檔案路徑 (.joblib)")
    parser.add_argument("--threshold", type=float, default=0.5, help="分類機率閾值 (預設 0.5)")
    args = parser.parse_args()

    detector = ClickbaitDetector(args.model)
    predictions = detector.predict(args.headlines, threshold=args.threshold)

    for item in predictions:
        print(
            f"[{item['label_name']:13s} | prob={item['clickbait_probability']:.4f}] "
            f"{item['headline']}"
        )


if __name__ == "__main__":
    main()
```

### 步驟 5.3：推論工具執行範例

```bash
# 單筆與多筆 CLI 推論測試
uv run python -m src.predict \
  "Which TV Female Friend Group Do You Belong In" \
  "Federal Reserve Announces Interest Rate Decision for Q4"
```
