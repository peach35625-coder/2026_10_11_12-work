# 監督式學習－標題黨偵測分類（Clickbait Classification）完整 Python 實作手冊

> **對應規劃文件**：`2026project/監督式學習分類完整專案規劃書.md`（v2.0 端到端 ML 流程）  
> **資料集路徑**：`clickbait_data.csv`（32,000 筆英文標題二元分類資料）  
> **Python 版本**：Python 3.12+  
> **核心設計原則**：全程使用 `sklearn.pipeline.Pipeline` 防資料洩漏、強制去偏前處理（小寫化與年份遮罩消融）、機率校準、切片錯誤分析與 Human-in-the-Loop 分級推論。

---

## Phase 0：環境建置與專案目錄初始化

### 步驟 0.1：建立資料夾架構與安裝套件

```bash
# 建立專案目錄結構
mkdir -p 2026project/notebooks 2026project/src 2026project/models 2026project/reports
cd 2026project

# 使用 uv 初始化 Python 3.12+ 環境並安裝相依套件
uv init --python 3.12
uv add pandas numpy scikit-learn lightgbm matplotlib seaborn joblib
```

### 步驟 0.2：建立套件初始化檔案 `src/__init__.py`

```python
# src/__init__.py
"""Clickbait Classification 專案核心模組（含去偏前處理、校準訓練、評估與監控）。"""

RANDOM_STATE = 42
```

---

## Phase 1：資料盤點、EDA 與偏差診斷（`notebooks/01_eda_and_bias_audit.py`）

對應規劃書 **Phase 1**：檢查基本品質、量化風格偏差（Title Case、`you/your`、數字開頭、標點失真、年份斷層），並繪製長度與 N-gram 分布。

```python
# notebooks/01_eda_and_bias_audit.py
from pathlib import Path
import re
import pandas as pd
import matplotlib.pyplot as plt
import seaborn as sns
from sklearn.feature_extraction.text import CountVectorizer

DATA_PATH = Path("clickbait_data.csv")
REPORT_DIR = Path("reports")
REPORT_DIR.mkdir(parents=True, exist_ok=True)

df = pd.read_csv(DATA_PATH)

print("=== 1. 資料集基礎盤點 ===")
print(f"總樣本數: {len(df):,}, 欄位: {df.columns.tolist()}")
print("缺失值:\n", df.isnull().sum())
print(f"完全重複列: {df.duplicated().sum()}")

norm_text = df["headline"].astype(str).str.lower().str.strip()
print(f"正規化（小寫+去空白）後重複數: {norm_text.duplicated().sum()}")
print("類別分布:\n", df["clickbait"].value_counts())

print("\n=== 2. 風格與格式偏差量化（Shortcut Audit） ===")
def audit_bias(sub_df: pd.DataFrame, label_name: str) -> dict:
    s = sub_df["headline"].astype(str)
    words = s.str.split().str.len()
    title_case_ratio = s.apply(
        lambda x: sum(1 for w in x.split() if w[:1].isupper()) / max(len(x.split()), 1) > 0.7
    ).mean()
    return {
        "類別": label_name,
        "樣本數": len(s),
        "平均字數": round(words.mean(), 2),
        "Title_Case比例": round(title_case_ratio, 3),
        "含第二人稱(you/your)比例": round(
            s.str.contains(r"\b(?:you|your|you're|yours)\b", case=False, regex=True).mean(), 3
        ),
        "數字開頭比例": round(s.str.match(r"^\d+").mean(), 3),
        "含問號比例": round(s.str.contains(r"\?", regex=True).mean(), 3),
        "含驚嘆號比例": round(s.str.contains(r"!", regex=True).mean(), 3),
        "含非ASCII比例": round(s.str.contains(r"[^\x00-\x7F]", regex=True).mean(), 3),
    }

audit_table = pd.DataFrame([
    audit_bias(df[df["clickbait"] == 0], "0_正常標題"),
    audit_bias(df[df["clickbait"] == 1], "1_標題黨"),
])
print(audit_table.to_string(index=False))

print("\n=== 3. 年份分布斷層檢查 ===")
for label in [0, 1]:
    years = (
        df.loc[df["clickbait"] == label, "headline"]
        .astype(str)
        .str.findall(r"\b(19\d{2}|20\d{2})\b")
        .explode()
        .dropna()
        .value_counts()
        .head(5)
    )
    print(f"類別 {label} Top-5 年份:", years.to_dict())

# 4. 繪製字數分布與 Top Bi-gram 比較圖
df["word_count"] = df["headline"].astype(str).str.split().str.len()
plt.figure(figsize=(8, 4))
sns.histplot(data=df, x="word_count", hue="clickbait", bins=26, kde=True, stat="density")
plt.title("Headline Word Count Distribution by Class")
plt.tight_layout()
plt.savefig(REPORT_DIR / "eda_word_count.png", dpi=150)
plt.close()

def get_top_ngrams(texts: pd.Series, ngram_range=(2, 2), top_k=15) -> pd.DataFrame:
    vec = CountVectorizer(ngram_range=ngram_range, stop_words="english", lowercase=True)
    mat = vec.fit_transform(texts)
    counts = mat.sum(axis=0).A1
    vocab = vec.get_feature_names_out()
    return (
        pd.DataFrame({"ngram": vocab, "count": counts})
        .sort_values("count", ascending=False)
        .head(top_k)
    )

print("\n標題黨 Top-10 Bi-gram:\n", get_top_ngrams(df[df["clickbait"] == 1]["headline"], top_k=10))
```

---

## Phase 2：資料讀取、正規化檢核與防漏切分（`src/dataset.py`）

對應規劃書 **Phase 2**：在切分前處理正規化重複，並以 `stratify=y` 嚴格切分為 **Train (70%) / Validation (15%) / Test (15%)**。

```python
# src/dataset.py
from pathlib import Path
from typing import Tuple
import pandas as pd
from sklearn.model_selection import train_test_split

from src import RANDOM_STATE


def load_and_clean_duplicates(csv_path: str | Path) -> pd.DataFrame:
    """讀取資料集並依小寫+去空白正規化移除重複標題，避免跨集合洩漏。"""
    df = pd.read_csv(csv_path)
    df = df.dropna(subset=["headline", "clickbait"]).copy()
    df["headline"] = df["headline"].astype(str).str.strip()
    df["clickbait"] = df["clickbait"].astype(int)

    # 依正規化文字去重（保留第一筆）
    df["_norm"] = df["headline"].str.lower().str.replace(r"\s+", " ", regex=True)
    dup_count = df["_norm"].duplicated().sum()
    if dup_count > 0:
        print(f"[Dataset] 偵測並移除 {dup_count} 筆正規化重複標題，防止跨集洩漏。")
        df = df.drop_duplicates(subset=["_norm"], keep="first")

    return df.drop(columns=["_norm"]).reset_index(drop=True)


![alt text](image.png)

---

## Phase 3：去偏前處理與多層次特徵工程（`src/features.py`）

對應規劃書 **Phase 3**：
- 強制轉小寫（剝除 Title Case 格式偏差）。
- 支援年份替換為 `<YEAR>` 權杖（供消融實驗驗證）。
- 結合詞級 TF-IDF（`ngram_range=(1, 2)`）＋ 字元級 TF-IDF（`char_wb` 降低 OOV 風險）＋ 結構統計特徵（字數、數字開頭、第二人稱；刻意排除 `?`/`!` 計數）。

```python
# src/features.py
import re
from typing import Iterable, List
import numpy as np
from sklearn.base import BaseEstimator, TransformerMixin
from sklearn.feature_extraction.text import TfidfVectorizer
from sklearn.pipeline import FeatureUnion, Pipeline
from sklearn.preprocessing import StandardScaler


class TextDebiasCleaner(BaseEstimator, TransformerMixin):
    """對標題執行去偏清理：強制小寫（可選關閉以做消融實驗）、空白正規化與年份遮罩。"""

    def __init__(self, lowercase: bool = True, mask_years: bool = True) -> None:
        self.lowercase = lowercase
        self.mask_years = mask_years

    def fit(self, X: Iterable[str], y=None):
        return self

    def transform(self, X: Iterable[str]) -> List[str]:
        cleaned = []
        for text in X:
            s = str(text).strip()
            s = re.sub(r"\s+", " ", s)
            if self.mask_years:
                s = re.sub(r"\b(19\d{2}|20\d{2})\b", "<YEAR>", s)
            if self.lowercase:
                s = s.lower()
            cleaned.append(s)
        return cleaned


class HeadlineStatsExtractor(BaseEstimator, TransformerMixin):
    """萃取安全之標題結構統計特徵（排除已失真之標點符號與大小寫比例）。"""

    _you_pattern = re.compile(r"\b(?:you|your|you're|yours)\b", re.IGNORECASE)
    _num_start_pattern = re.compile(r"^\d+")

    def fit(self, X: Iterable[str], y=None):
        return self

    def transform(self, X: Iterable[str]) -> np.ndarray:
        feats = []
        for text in X:
            s = str(text).strip()
            words = s.split()
            word_count = len(words)
            char_count = len(s)
            starts_with_num = 1.0 if self._num_start_pattern.match(s) else 0.0
            has_second_person = 1.0 if self._you_pattern.search(s) else 0.0
            feats.append([word_count, char_count, starts_with_num, has_second_person])
        return np.asarray(feats, dtype=np.float64)


def build_feature_union(
    lowercase: bool = True,
    mask_years: bool = True,
    word_max_features: int = 20000,
    char_max_features: int = 15000,
    min_df: int = 3,
    use_char_ngrams: bool = True,
    use_stats: bool = True,
) -> FeatureUnion:
    """建構防洩漏之多層次特徵聯集管線。"""
    word_pipe = Pipeline([
        ("cleaner", TextDebiasCleaner(lowercase=lowercase, mask_years=mask_years)),
        ("tfidf_word", TfidfVectorizer(
            lowercase=False,  # 已由 TextDebiasCleaner 控制，方便進行消融實驗
            ngram_range=(1, 2),
            min_df=min_df,
            max_features=word_max_features,
            sublinear_tf=True,
        )),
    ])

    transformers = [("word_tfidf", word_pipe)]

    if use_char_ngrams:
        char_pipe = Pipeline([
            ("cleaner", TextDebiasCleaner(lowercase=lowercase, mask_years=mask_years)),
            ("tfidf_char", TfidfVectorizer(
                lowercase=False,
                analyzer="char_wb",
                ngram_range=(3, 5),
                min_df=5,
                max_features=char_max_features,
                sublinear_tf=True,
            )),
        ])
        transformers.append(("char_tfidf", char_pipe))

    if use_stats:
        stats_pipe = Pipeline([
            ("stats", HeadlineStatsExtractor()),
            ("scaler", StandardScaler()),
        ])
        transformers.append(("stats", stats_pipe))

    return FeatureUnion(transformer_list=transformers)
```

---

## Phase 4：模型訓練、消融實驗、調參與機率校準（`src/train.py`）

對應規劃書 **Phase 4**：
1. 進行「大小寫與年份遮罩」消融實驗，確認效能並非靠格式捷徑。
2. 比較 `MultinomialNB`（純詞級）、`LogisticRegression`、`LinearSVC`（經 `CalibratedClassifierCV` 校準）、`LightGBM`（`TruncatedSVD` + 統計特徵）。
3. 使用 5-Fold Stratified CV 調參，並於**驗證集**以 PR 曲線尋找最佳決策閾值，最後將最佳模型連同元資料存為 `.joblib`。

```python
# src/train.py
from pathlib import Path
from typing import Dict, Any, Tuple
import joblib
import numpy as np
import pandas as pd
from lightgbm import LGBMClassifier
from sklearn.calibration import CalibratedClassifierCV
from sklearn.decomposition import TruncatedSVD
from sklearn.linear_model import LogisticRegression
from sklearn.metrics import f1_score, roc_auc_score, average_precision_score, precision_recall_curve
from sklearn.model_selection import GridSearchCV, StratifiedKFold
from sklearn.naive_bayes import MultinomialNB
from sklearn.pipeline import Pipeline
from sklearn.svm import LinearSVC

from src import RANDOM_STATE
from src.dataset import load_and_clean_duplicates, split_dataset
from src.features import build_feature_union, TextDebiasCleaner
from sklearn.feature_extraction.text import TfidfVectorizer

MODEL_DIR = Path("models")
MODEL_DIR.mkdir(parents=True, exist_ok=True)


def evaluate_probs(y_true: np.ndarray, y_prob: np.ndarray, threshold: float = 0.5) -> Dict[str, float]:
    y_pred = (y_prob >= threshold).astype(int)
    return {
        "f1": round(f1_score(y_true, y_pred), 4),
        "roc_auc": round(roc_auc_score(y_true, y_prob), 4),
        "pr_auc": round(average_precision_score(y_true, y_prob), 4),
    }


def run_ablation_study(train_df: pd.DataFrame, val_df: pd.DataFrame) -> pd.DataFrame:
    """消融實驗：對比大小寫與年份遮罩對驗證集效能的影響。"""
    configs = [
        {"name": "1_保留大小寫_保留年份", "lowercase": False, "mask_years": False},
        {"name": "2_強制小寫_保留年份", "lowercase": True, "mask_years": False},
        {"name": "3_強制小寫_年份遮罩(去偏推薦)", "lowercase": True, "mask_years": True},
    ]
    rows = []
    for cfg in configs:
        pipe = Pipeline([
            ("features", build_feature_union(lowercase=cfg["lowercase"], mask_years=cfg["mask_years"])),
            ("clf", LogisticRegression(max_iter=1000, C=1.0, random_state=RANDOM_STATE)),
        ])
        pipe.fit(train_df["headline"], train_df["clickbait"])
        probs = pipe.predict_proba(val_df["headline"])[:, 1]
        metrics = evaluate_probs(val_df["clickbait"].to_numpy(), probs)
        rows.append({"實驗設定": cfg["name"], **metrics})
    return pd.DataFrame(rows)


def find_optimal_threshold(y_true: np.ndarray, y_prob: np.ndarray) -> Tuple[float, float]:
    """依驗證集 Precision-Recall 曲線尋找 F1 最大化之決策閾值。"""
    precisions, recalls, thresholds = precision_recall_curve(y_true, y_prob)
    f1s = 2 * precisions[:-1] * recalls[:-1] / np.clip(precisions[:-1] + recalls[:-1], 1e-12, None)
    best_idx = int(np.argmax(f1s))
    return float(thresholds[best_idx]), float(f1s[best_idx])


def main() -> None:
    df = load_and_clean_duplicates("clickbait_data.csv")
    train_df, val_df, test_df = split_dataset(df)

    X_train, y_train = train_df["headline"], train_df["clickbait"].to_numpy()
    X_val, y_val = val_df["headline"], val_df["clickbait"].to_numpy()

    print("=== 1. 執行去偏消融實驗 ===")
    ablation_df = run_ablation_study(train_df, val_df)
    print(ablation_df.to_string(index=False))

    print("\n=== 2. 多模型訓練與比較（強制去偏設定：lowercase=True, mask_years=True） ===")
    cv = StratifiedKFold(n_splits=5, shuffle=True, random_state=RANDOM_STATE)

    # 候選 1：Baseline MultinomialNB（僅詞級 TF-IDF，因 NB 不可輸入標準化負值）
    nb_pipe = Pipeline([
        ("cleaner", TextDebiasCleaner(lowercase=True, mask_years=True)),
        ("tfidf", TfidfVectorizer(lowercase=False, ngram_range=(1, 2), min_df=3, sublinear_tf=True)),
        ("clf", MultinomialNB(alpha=0.5)),
    ])
    nb_pipe.fit(X_train, y_train)
    nb_metrics = evaluate_probs(y_val, nb_pipe.predict_proba(X_val)[:, 1])
    print("MultinomialNB (Val):", nb_metrics)

    # 候選 2：LogisticRegression + GridSearchCV
    lr_pipe = Pipeline([
        ("features", build_feature_union(lowercase=True, mask_years=True)),
        ("clf", LogisticRegression(max_iter=1000, random_state=RANDOM_STATE)),
    ])
    lr_grid = GridSearchCV(
        lr_pipe,
        param_grid={"clf__C": [0.5, 1.0, 2.0, 5.0]},
        cv=cv,
        scoring="f1",
        n_jobs=-1,
    )
    lr_grid.fit(X_train, y_train)
    lr_best = lr_grid.best_estimator_
    lr_probs = lr_best.predict_proba(X_val)[:, 1]
    print(f"LogisticRegression (Best C={lr_grid.best_params_['clf__C']}, Val):", evaluate_probs(y_val, lr_probs))

    # 候選 3：LinearSVC + Platt Scaling 機率校準
    svc_pipe = Pipeline([
        ("features", build_feature_union(lowercase=True, mask_years=True)),
        ("clf", CalibratedClassifierCV(
            estimator=LinearSVC(C=1.0, random_state=RANDOM_STATE),
            method="sigmoid",
            cv=3,
        )),
    ])
    svc_pipe.fit(X_train, y_train)
    svc_probs = svc_pipe.predict_proba(X_val)[:, 1]
    print("Calibrated LinearSVC (Val):", evaluate_probs(y_val, svc_probs))

    # 候選 4：LightGBM（TF-IDF + SVD 降維）
    lgbm_pipe = Pipeline([
        ("features", build_feature_union(lowercase=True, mask_years=True, use_char_ngrams=False)),
        ("svd", TruncatedSVD(n_components=150, random_state=RANDOM_STATE)),
        ("clf", LGBMClassifier(
            n_estimators=300,
            learning_rate=0.05,
            num_leaves=31,
            random_state=RANDOM_STATE,
            verbose=-1,
        )),
    ])
    lgbm_pipe.fit(X_train, y_train)
    lgbm_probs = lgbm_pipe.predict_proba(X_val)[:, 1]
    print("LightGBM + SVD (Val):", evaluate_probs(y_val, lgbm_probs))

    # 依驗證集 PR-AUC / F1 選出最佳模型（以 lr_best 與 svc_pipe 擇優示範）
    candidates: Dict[str, Tuple[Any, np.ndarray]] = {
        "LogisticRegression": (lr_best, lr_probs),
        "CalibratedLinearSVC": (svc_pipe, svc_probs),
        "LightGBM_SVD": (lgbm_pipe, lgbm_probs),
    }
    best_name, (best_model, best_val_probs) = max(
        candidates.items(),
        key=lambda item: f1_score(y_val, (item[1][1] >= 0.5).astype(int)),
    )

    opt_thresh, opt_f1 = find_optimal_threshold(y_val, best_val_probs)
    print(f"\n最佳候選模型: {best_name} | 驗證集最佳閾值: {opt_thresh:.4f} (F1={opt_f1:.4f})")

    bundle = {
        "model_name": best_name,
        "pipeline": best_model,
        "threshold": opt_thresh,
        "val_baseline_probs": best_val_probs,  # 供 Phase 7 計算 PSI 分布漂移
    }
    joblib.dump(bundle, MODEL_DIR / "clickbait_best_bundle.joblib")
    print(f"已儲存模型包至: {MODEL_DIR / 'clickbait_best_bundle.joblib'}")


if __name__ == "__main__":
    main()
```

---

## Phase 5：模型評估、可解釋性審查與分層錯誤分析（`src/evaluate.py`）

對應規劃書 **Phase 5**：
1. 在隔離測試集上計算完整指標與單筆推論延遲（`< 20 ms`）。
2. 匯出權重最高的前 30 個正負向詞彙（檢核是否被年代或明星詞主導）。
3. **分層錯誤分析（Sliced Error Analysis）**：針對「含數字正常新聞」、「不含 `you` 的標題黨」、「極短標題（$\le 4$ 字）」檢視誤判率。
4. **Out-of-Time 外部近期標題盲測**：實測模型面對 2024–2026 年現代標點（`?`、`!`）與新詞彙的表現。

```python
# src/evaluate.py
import time
from pathlib import Path
import joblib
import numpy as np
import pandas as pd
import matplotlib.pyplot as plt
import seaborn as sns
from sklearn.metrics import (
    accuracy_score,
    average_precision_score,
    classification_report,
    confusion_matrix,
    f1_score,
    log_loss,
    precision_score,
    recall_score,
    roc_auc_score,
)

from src.dataset import load_and_clean_duplicates, split_dataset

MODEL_PATH = Path("models/clickbait_best_bundle.joblib")
REPORT_DIR = Path("reports")
REPORT_DIR.mkdir(parents=True, exist_ok=True)


def inspect_top_features(pipeline, top_k: int = 30) -> None:
    """若最終分類器為 LogisticRegression，輸出權重最高之正負特徵詞進行可解釋性審查。"""
    clf = pipeline.named_steps.get("clf")
    if not hasattr(clf, "coef_"):
        print("[Explainability] 目前最佳模型為校準封裝或非線性模型，略過線性權重直接展開。")
        return

    features = pipeline.named_steps["features"]
    word_vec = features.transformer_list[0][1].named_steps["tfidf_word"]
    word_names = [f"word:{w}" for w in word_vec.get_feature_names_out()]

    # 若僅看詞級特徵前段權重（word_tfidf 位於 FeatureUnion 最前方）
    coefs = clf.coef_[0][: len(word_names)]
    top_pos_idx = np.argsort(coefs)[-top_k:][::-1]
    top_neg_idx = np.argsort(coefs)[:top_k]

    print(f"\n=== 可解釋性審查：Top-{top_k} 標題黨正向特徵詞 ===")
    print(", ".join(f"{word_names[i]}({coefs[i]:.2f})" for i in top_pos_idx[:15]))
    print(f"=== 可解釋性審查：Top-{top_k} 正常標題負向特徵詞 ===")
    print(", ".join(f"{word_names[i]}({coefs[i]:.2f})" for i in top_neg_idx[:15]))


def sliced_error_analysis(test_df: pd.DataFrame, y_pred: np.ndarray) -> pd.DataFrame:
    """分層切片錯誤分析：檢查高風險子群體的誤判情況。"""
    df_eval = test_df.copy()
    df_eval["y_pred"] = y_pred
    df_eval["is_error"] = df_eval["clickbait"] != df_eval["y_pred"]
    s = df_eval["headline"].astype(str)

    slices = {
        "1_含數字之正常新聞(FP風險)": (df_eval["clickbait"] == 0) & s.str.contains(r"\b\d+\b", regex=True),
        "2_不含you之標題黨(FN風險)": (df_eval["clickbait"] == 1) & ~s.str.contains(r"\b(?:you|your|you're|yours)\b", case=False, regex=True),
        "3_極短標題(<=4字)": s.str.split().str.len() <= 4,
    }

    rows = []
    for name, mask in slices.items():
        sub = df_eval[mask]
        err_rate = sub["is_error"].mean() if len(sub) > 0 else 0.0
        rows.append({
            "切片群體": name,
            "樣本數": len(sub),
            "錯誤筆數": int(sub["is_error"].sum()),
            "錯誤率": f"{err_rate:.2%}",
        })
    return pd.DataFrame(rows)


def main() -> None:
    bundle = joblib.load(MODEL_PATH)
    pipeline = bundle["pipeline"]
    threshold = bundle["threshold"]

    df = load_and_clean_duplicates("clickbait_data.csv")
    _, _, test_df = split_dataset(df)
    X_test, y_test = test_df["headline"], test_df["clickbait"].to_numpy()

    # 實測單筆推論延遲
    sample_list = X_test.iloc[:200].tolist()
    t0 = time.perf_counter()
    for text in sample_list:
        _ = pipeline.predict_proba([text])
    avg_latency_ms = ((time.perf_counter() - t0) / len(sample_list)) * 1000

    # 測試集完整評估
    y_prob = pipeline.predict_proba(X_test)[:, 1]
    y_pred = (y_prob >= threshold).astype(int)

    print(f"=== 隔離測試集最終評估（模型: {bundle['model_name']}, 閾值: {threshold:.4f}） ===")
    print(f"Accuracy : {accuracy_score(y_test, y_pred):.4f}")
    print(f"Precision: {precision_score(y_test, y_pred):.4f}")
    print(f"Recall   : {recall_score(y_test, y_pred):.4f}")
    print(f"F1-Score : {f1_score(y_test, y_pred):.4f}")
    print(f"ROC-AUC  : {roc_auc_score(y_test, y_prob):.4f}")
    print(f"PR-AUC   : {average_precision_score(y_test, y_prob):.4f}")
    print(f"Log Loss : {log_loss(y_test, y_prob):.4f}")
    print(f"平均單筆推論延遲: {avg_latency_ms:.3f} ms")
    print("\n分類報告:\n", classification_report(y_test, y_pred, target_names=["正常標題(0)", "標題黨(1)"]))

    # 儲存混淆矩陣圖
    cm = confusion_matrix(y_test, y_pred)
    plt.figure(figsize=(5, 4))
    sns.heatmap(cm, annot=True, fmt="d", cmap="Blues", xticklabels=["Pred 0", "Pred 1"], yticklabels=["True 0", "True 1"])
    plt.title(f"Confusion Matrix ({bundle['model_name']})")
    plt.tight_layout()
    plt.savefig(REPORT_DIR / "test_confusion_matrix.png", dpi=150)
    plt.close()

    inspect_top_features(pipeline, top_k=30)

    print("\n=== 分層切片錯誤分析 ===")
    print(sliced_error_analysis(test_df, y_pred).to_string(index=False))

    # 外部近期資料盲測（Out-of-Time 模擬樣本：含現代標點、AI 話題與新聞數字）
    oot_samples = [
        ("Federal Reserve Holds Interest Rates Steady Amid Inflation Data", 0),
        ("You Won't Believe What This AI Tool Just Did To My Workflow?!", 1),
        ("2026 Global EV Sales Rise 18 Percent in Q1 Report", 0),
        ("7 Shocking Habits That Quietly Ruin Your Sleep Every Night!", 1),
    ]
    oot_texts = [t for t, _ in oot_samples]
    oot_probs = pipeline.predict_proba(oot_texts)[:, 1]
    print("\n=== 外部近期標題（Out-of-Time）盲測 ===")
    for (text, true_lbl), p in zip(oot_samples, oot_probs):
        print(f"[真實:{true_lbl} | 預測機率:{p:.3f}] {text}")


if __name__ == "__main__":
    main()
```

---

## Phase 6：模型封裝與 Human-in-the-Loop 分級推論（`src/predict.py`）

對應規劃書 **Phase 6**：提供 CLI 與函式推論介面，依據校準機率實作三級決策（高信心標題黨、灰色邊界人工複審、正常標題），降低誤殺正當內容風險。

```python
# src/predict.py
import argparse
from pathlib import Path
from typing import Dict, List
import joblib

DEFAULT_MODEL_PATH = Path("models/clickbait_best_bundle.joblib")


def classify_with_hitl(
    headlines: List[str],
    model_path: str | Path = DEFAULT_MODEL_PATH,
    high_conf_cutoff: float = 0.85,
    review_low_cutoff: float = 0.40,
) -> List[Dict]:
    """
    Human-in-the-Loop 分級推論：
    - P >= 0.85: high_confidence_clickbait（建議標記或降權）
    - 0.40 <= P < 0.85: human_review_required（灰色地帶，轉人工複審）
    - P < 0.40: non_clickbait（正常放行）
    """
    bundle = joblib.load(model_path)
    pipeline = bundle["pipeline"]
    probs = pipeline.predict_proba(headlines)[:, 1]

    results = []
    for text, p in zip(headlines, probs):
        p_val = float(p)
        if p_val >= high_conf_cutoff:
            decision = "high_confidence_clickbait"
        elif p_val >= review_low_cutoff:
            decision = "human_review_required"
        else:
            decision = "non_clickbait"

        results.append({
            "headline": text,
            "clickbait_probability": round(p_val, 4),
            "decision": decision,
        })
    return results


if __name__ == "__main__":
    parser = argparse.ArgumentParser(description="標題黨偵測分級推論工具")
    parser.add_argument("headlines", nargs="+", help="輸入一個或多個英文標題")
    args = parser.parse_args()

    for res in classify_with_hitl(args.headlines):
        print(res)
```

---

## Phase 7：上線概念漂移監控與重訓觸發（`src/monitor.py`）

對應規劃書 **Phase 7**：計算線上輸入標題之預測機率分布與驗證集基線間的 **PSI（Population Stability Index）**，當 `PSI >= 0.20` 時發出概念漂移警示並觸發抽樣標註與重訓流程。

```python
# src/monitor.py
from pathlib import Path
import joblib
import numpy as np

MODEL_PATH = Path("models/clickbait_best_bundle.joblib")


def calculate_psi(expected: np.ndarray, actual: np.ndarray, bins: int = 10) -> float:
    """計算 Population Stability Index (PSI) 監控預測機率分布漂移。"""
    bin_edges = np.linspace(0.0, 1.0, bins + 1)
    expected_counts, _ = np.histogram(expected, bins=bin_edges)
    actual_counts, _ = np.histogram(actual, bins=bin_edges)

    expected_pct = np.clip(expected_counts / max(len(expected), 1), 1e-6, None)
    actual_pct = np.clip(actual_counts / max(len(actual), 1), 1e-6, None)

    psi = np.sum((actual_pct - expected_pct) * np.log(actual_pct / expected_pct))
    return float(psi)


def check_production_drift(new_headlines: list[str], psi_alert_threshold: float = 0.20) -> dict:
    """檢查新一批線上標題是否發生顯著概念漂移。"""
    bundle = joblib.load(MODEL_PATH)
    baseline_probs = bundle["val_baseline_probs"]
    new_probs = bundle["pipeline"].predict_proba(new_headlines)[:, 1]

    psi_value = calculate_psi(baseline_probs, new_probs)
    status = "ALERT_RETRAIN_NEEDED" if psi_value >= psi_alert_threshold else "STABLE"

    return {
        "sample_size": len(new_headlines),
        "psi": round(psi_value, 4),
        "status": status,
    }


if __name__ == "__main__":
    sample_prod_batch = [
        "10 New AI Frameworks Developers Are Switching To in 2026",
        "Central Bank Announces Quarterly Economic Outlook",
        "Why Everyone Is Talking About This Viral Coffee Recipe",
    ] * 20
    print(check_production_drift(sample_prod_batch))
```
