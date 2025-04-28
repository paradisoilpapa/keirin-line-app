# ライン競輪 7車ライン＋政春補正＋欠番対応版（Streamlitアプリ版）
import streamlit as st
import pandas as pd

st.title("ライン競輪スコア計算（7車ライン＋政春補正＋欠番対応）")

kakushitsu_options = ['逃', '両', '追']
symbol_input_options = ['◎', '〇', '▲', '△', '×', '無']

st.header("【脚質入力】")
kakushitsu = [st.selectbox(f"{i+1}番脚質", kakushitsu_options, key=f"kaku_{i}") for i in range(7)]

st.header("【前走着順入力（1〜7着）】")
chaku = [st.number_input(f"{i+1}番着順", min_value=1, max_value=7, value=5, step=1, key=f"chaku_{i}") for i in range(7)]

st.header("【競争得点入力】")
rating = [st.number_input(f"{i+1}番得点", value=55.0, step=0.1, key=f"rate_{i}") for i in range(7)]

st.header("【隊列順位入力（数字、欠の場合は空欄）】")
tairetsu = [st.text_input(f"{i+1}番隊列順位", key=f"tai_{i}") for i in range(7)]

st.header("【ラインポジション入力（0単騎 1先頭 2番手 3三番手）】")
line_order = [st.selectbox(f"{i+1}番ラインポジション", [0, 1, 2, 3], key=f"line_{i}") for i in range(7)]

st.header("【政春印入力】")
symbol = [st.selectbox(f"{i+1}番印", symbol_input_options, key=f"sym_{i}") for i in range(7)]

st.header("【バンク・風条件】")
wind_direction = st.selectbox("風向", ['無風', '向かい風', '追い風', 'ホーム向かい風', 'ホーム追い風'])
wind_speed = st.number_input("風速(m/s)", min_value=0.0, max_value=10.0, step=0.1)
straight_length = st.number_input("みなし直線(m)", min_value=30, max_value=80, value=52, step=1)
bank_angle = st.number_input("バンク角(°)", min_value=20.0, max_value=45.0, value=30.0, step=0.1)
rain = st.checkbox("雨（滑走・慎重傾向あり）")

if st.button("スコア計算実行"):
    base_score = {'逃': 8, '両': 6, '追': 5}
    symbol_bonus = {'◎': 2.0, '〇': 1.5, '▲': 1.0, '△': 0.5, '×': 0.2, '無': 0.0}

    def wind_straight_combo_adjust(kaku, direction, speed, straight):
        if speed < 0.5:
            return 0
        if direction == "ホーム向かい風":
            if straight <= 50:
                return {'逃': +3, '両': 0, '追': -2}[kaku]
            elif straight <= 58:
                return {'逃': -1, '両': +1, '追': +1}[kaku]
            else:
                return {'逃': -2, '両': +1.5, '追': +3}[kaku]
        elif direction == "追い風":
            return {'逃': -1, '両': 0, '追': +1}[kaku]
        elif direction == "向かい風":
            return {'逃': +1, '両': 0, '追': -1}[kaku]
        return 0

    def tairyetsu_adjust(num, tairetsu_list):
        pos = tairetsu_list.index(num)
        base = max(0, round(3.0 - 0.5 * pos, 1))
        if kakushitsu[num - 1] == '追':
            if 2 <= pos <= 4:
                return base + 0.5 + 1.5
            else:
                return base + 0.5
        return base

    def score_from_chakujun(pos):
        if pos == 1: return 3.0
        elif pos == 2: return 2.5
        elif pos == 3: return 2.0
        elif pos <= 6: return 1.0
        else: return 0.0

    def rain_adjust(kaku):
        if not rain:
            return 0
        return {'逃': +2.5, '両': +0.5, '追': -2.5}[kaku]

    def line_member_bonus(pos):
        if pos == 0:
            return -1.0
        elif pos == 1:
            return 2.0
        elif pos == 2:
            return 1.5
        elif pos == 3:
            return 1.0
        return 0.0

    def bank_character_bonus(kaku, angle, straight):
        if straight <= 50 and angle >= 32:
            return {'逃': +1.0, '両': 0, '追': -1.0}[kaku]
        elif straight >= 58 and angle <= 31:
            return {'逃': -1.0, '両': 0, '追': +1.0}[kaku]
        return 0.0

    tairetsu_list = [i+1 for i, v in enumerate(tairetsu) if v.isdigit()]

    score_parts = []
    for i in range(7):
        if not tairetsu[i].isdigit():
            continue
        num = i + 1
        base = base_score[kakushitsu[i]]
        wind = wind_straight_combo_adjust(kakushitsu[i], wind_direction, wind_speed, straight_length)
        tai = tairyetsu_adjust(num, tairetsu_list)
        kasai = score_from_chakujun(chaku[i])
        rating_score = max(0, round((sum(rating)/7 - rating[i]) * 0.2, 1))
        rain_corr = rain_adjust(kakushitsu[i])
        symbol_bonus_score = symbol_bonus[symbol[i]]
        line_bonus = line_member_bonus(line_order[i])
        bank_bonus = bank_character_bonus(kakushitsu[i], bank_angle, straight_length)
        total = base + wind + tai + kasai + rating_score + rain_corr + symbol_bonus_score + line_bonus + bank_bonus

        score_parts.append((num, kakushitsu[i], base, wind, tai, kasai, rating_score, rain_corr, symbol_bonus_score, line_bonus, bank_bonus, total))

    df = pd.DataFrame(score_parts, columns=['車番', '脚質', '基本', '風補正', '隊列補正', '着順補正', '得点補正', '雨補正', '政春印補正', 'ライン補正', 'バンク補正', '合計スコア'])
    df_sorted = df.sort_values(by='合計スコア', ascending=False).reset_index(drop=True)

    st.dataframe(df_sorted)
