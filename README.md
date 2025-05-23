SAFE-ALPHA: Fundamentals-Based Stock Predictor with NSE Data & Real-Time News 

import pandas as pd import streamlit as st import requests from bs4 import BeautifulSoup from datetime import datetime

st.set_page_config(page_title="SAFE-ALPHA", layout="wide") st.title("SAFE-ALPHA: Indian Stock Fundamental Predictor")

@st.cache_data(show_spinner=True) def fetch_fundamentals(stock): url = f"https://www.screener.in/company/{stock}/" try: res = requests.get(url, headers={"User-Agent": "Mozilla/5.0"}) soup = BeautifulSoup(res.text, 'html.parser')

def extract_float(text): try: return float(text.replace('%', '').replace(',', '').strip()) except: return None try: pe = extract_float(soup.find('li', text=lambda t: 'P/E' in t).text.split(':')[-1]) except: pe = None ratios = soup.select(".company-ratios .flex.flex-space-between") data = {r.select_one('span').text.strip(): r.select_one('span+span').text.strip() for r in ratios} return { 'Stock': stock, 'PE': pe, 'Sector_PE': None, 'ROE': extract_float(data.get('ROE', '0')), 'ROCE': extract_float(data.get('ROCE', '0')), 'DebtEquity': extract_float(data.get('Debt to equity', '0')), 'FCF_Yield': 1, 'ProfitGrowth5Y': extract_float(data.get('Profit growth', '0')), 'PEG': 0.9, 'Promoter_Holding_Trend': 'Increasing', 'Dividend_Yield': extract_float(data.get('Dividend Yield', '0')) } except: return None 

def calculate_score(row): score = 0 if row['PE'] and row['Sector_PE'] and row['PE'] < row['Sector_PE']: score += 10 if row['ROE'] and row['ROE'] >= 18: score += 15 if row['ROCE'] and row['ROCE'] >= 18: score += 15 if row['DebtEquity'] is not None and row['DebtEquity'] < 0.5: score += 10 if row['FCF_Yield'] > 0: score += 10 if row['ProfitGrowth5Y'] and row['ProfitGrowth5Y'] >= 12: score += 15 if row['PEG'] and row['PEG'] < 1: score += 10 if row['Promoter_Holding_Trend'] == 'Increasing': score += 5 if row['Dividend_Yield'] and row['Dividend_Yield'] > 1: score += 10 return score

@st.cache_data(show_spinner=True) def fetch_news(stock): news_url = f"https://news.google.com/search?q={stock}+stock+when:7d&hl=en-IN&gl=IN&ceid=IN:en" res = requests.get(news_url, headers={"User-Agent": "Mozilla/5.0"}) soup = BeautifulSoup(res.text, 'html.parser') headlines = soup.select("article h3") return [h.get_text() for h in headlines[:5]]

stocks = st.text_area("Enter NSE stock symbols (comma-separated, e.g. INFY, TCS, HDFC)")

if stocks: symbols = [s.strip().upper() for s in stocks.split(',')] data = [] with st.spinner("Fetching data from Screener.in and Google News..."): for symbol in symbols: fundamentals = fetch_fundamentals(symbol) if fundamentals: fundamentals['News'] = fetch_news(symbol) data.append(fundamentals)

if data: df = pd.DataFrame(data) df['Sector_PE'] = df['PE'].mean() df['Score'] = df.apply(calculate_score, axis=1) df['Recommendation'] = df['Score'].apply(lambda x: 'Buy' if x >= 75 else 'Avoid') st.subheader("Stock Scores") st.dataframe(df[['Stock', 'Score', 'Recommendation']]) for d in data: st.markdown(f"### {d['Stock']} News") for news in d['News']: st.markdown(f"- {news}") csv = df.to_csv(index=False).encode('utf-8') st.download_button("Download Results as CSV", data=csv, file_name="safe_alpha_scores.csv", mime="text/csv") else: st.error("No data found. Try using NSE symbols like INFY, TCS, HDFCBANK, etc.") 
