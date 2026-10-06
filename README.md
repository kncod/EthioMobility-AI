# Addis Ride Demand Forecasting — Addis Demand AI

Qiyas AI Hackathon #2: Addis Ride Demand Forecasting Challenge  
**Track:** Intelligent Data & AI Engineering (IDAE)  
**Program:** EDI-Qiyas-CoDiST Advanced Digital Skills Program

## Team

| | |
|--|--|
| **Team name** | Addis Demand AI |
| **Members** | Abraham Getachew, Ahmed Hussen, Natanim Masresha, Nigus Shiferaw, Tekilu Asefa |

## Setup

```bash
cd c:\xampp\ride_forcasting
python -m venv .venv
.venv\Scripts\activate
pip install -r requirements.txt
```

## Run order

1. **Cleaning & integration (A)** — `python src/run_pipeline.py` or `notebooks/01_cleaning_and_integration.ipynb`
2. **Analysis (B)** — `python src/analysis.py` or `notebooks/02_analysis_report.ipynb`
3. **Figures (C)** — `python src/visualizations.py` or `notebooks/03_visualizations.ipynb`
4. **Modeling (D)** — `python src/modeling.py` or `notebooks/04_modeling_and_evaluation.ipynb`
5. **Demo (E)** — `python -m streamlit run app/app.py`
6. **Slides (F)** — `presentation/team_addis_demand_ai_slides.pptx` (`python src/make_slides.py`)

## Deliverables

- Raw CSVs in `data/raw/` (never edited)
- Processed: `data/processed/master_train.csv`, `master_test.csv`, `data_dictionary_master.csv`
- Reports: `reports/A_*.md`, `B_analysis_report.md`, `D_model_evaluation.md`
- Figures: `figures/fig01`–`fig12` + `figure_captions.md`
- Model: `models/final_model.joblib`
- Submission: `submission/team_addis_demand_ai_submission.csv` (4,032 rows)
- Demo: `app/app.py` (local: http://localhost:8501)
- Slides: `presentation/team_addis_demand_ai_slides.pptx`

## Validation score (chronological, 18–31 Oct)

- **HistGBM RMSE ≈ 10.16** · MAE ≈ 6.41
- Seasonal naive RMSE ≈ 12.08 · Mean baseline ≈ 27.88

## Demo

```bash
python -m streamlit run app/app.py
```

Pick a zone and any date in **1–14 Nov 2025**. Weather/events load automatically.  
Dashboard includes uncertainty band, vs-typical %, city overview tab, and CSV download.  
Local demo is fine if hosting is unavailable.

## Notes

- Timestamps after cleaning use **Africa/Addis_Ababa**.
- Model excludes `avg_fare_birr`, `avg_wait_min`, `active_drivers` (not known at forecast time).
- Validation is chronological — never random split, never score on the test file.
