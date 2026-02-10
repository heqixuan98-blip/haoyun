import streamlit as st
import time
import random
import json
import os
from datetime import datetime, timedelta

# ================= 1. 全局配置与样式 =================

st.set_page_config(
    page_title="好孕签 · 四季孕育助手",
    page_icon="🏮",
    layout="wide",
    initial_sidebar_state="expanded"
)

st.markdown("""
    <style>
    .stApp {
        background-color: #fdfbf7;
        font-family: "PingFang SC", "Microsoft YaHei", sans-serif;
    }
    
    /* 大卡片样式 */
    .dashboard-card {
        background: white;
        padding: 20px;
        border-radius: 15px;
        box-shadow: 0 4px 15px rgba(0,0,0,0.05);
        margin-bottom: 15px;
        border: 1px solid #f0f0f0;
    }
    
    /* 季节卡片 - 动态变化背景 */
    .season-card {
        background: linear-gradient(135deg, #e0f7fa 0%, #ffffff 100%); /* 默认冬/春色 */
        padding: 15px;
        border-radius: 15px;
        border-left: 5px solid #00bcd4;
        margin-bottom: 20px;
    }
    
    /* 食谱卡片 */
    .recipe-box {
        background: #fff;
        padding: 15px;
        border-radius: 12px;
        border: 1px solid #eee;
        margin-bottom: 10px;
        transition: transform 0.2s;
    }
    .recipe-box:hover {
        transform: translateY(-2px);
        box-shadow: 0 5px 15px rgba(0,0,0,0.08);
    }
    
    .tag {
        display: inline-block;
        padding: 2px 8px;
        border-radius: 5px;
        font-size: 12px;
        margin-right: 5px;
    }
    .tag-week { background: #ffebee; color: #c62828; }
    .tag-season { background: #e8f5e9; color: #2e7d32; }

    .footer { text-align: center; color: #aaa; font-size: 12px; margin-top: 50px; }
    </style>
""", unsafe_allow_html=True)

# ================= 2. 数据层：孕周 + 四季 =================

# --- A. 季节时令库 (根据月份推荐) ---
SEASONAL_DB = {
    "spring": {
        "months": [3, 4, 5],
        "name": "🌱 春季 (万物生发)",
        "desc": "春季细菌活跃，注意预防流感。饮食宜清淡，少酸多甘，养肝健脾。",
        "eat": ["🍓 草莓 (维C之王)", "🍒 樱桃 (补铁)", "🥬 菠菜 (补叶酸)", "🎍 春笋 (通便)"],
        "tip": "春捂秋冻，不要脱衣服太快哦！"
    },
    "summer": {
        "months": [6, 7, 8],
        "name": "☀️ 夏季 (清热解暑)",
        "desc": "天气炎热，出汗多，注意补充水分和电解质。切贪凉，少吃冰镇西瓜。",
        "eat": ["🍅 西红柿 (防晒)", "🥒 黄瓜 (补水)", "🍑 桃子 (养人)", "🍵 绿豆汤 (解暑)"],
        "tip": "吹空调睡觉记得盖住肚子，防止着凉宫缩。"
    },
    "autumn": {
        "months": [9, 10, 11],
        "name": "🍂 秋季 (滋阴润肺)",
        "desc": "秋燥易伤肺，皮肤容易干痒。多吃白色食物，滋阴润燥。",
        "eat": ["🍐 秋梨 (润肺)", "🍇 葡萄 (抗氧化)", "🌰 板栗 (补肾)", "🍵 银耳羹 (润肤)"],
        "tip": "早晚温差大，出门带件薄外套。"
    },
    "winter": {
        "months": [12, 1, 2],
        "name": "❄️ 冬季 (补肾藏精)",
        "desc": "寒气重，宜温补。注意手脚保暖，多晒太阳促进钙吸收。",
        "eat": ["🥘 萝卜 (小人参)", "🥩 羊肉 (温补/适量)", "🍠 红薯 (暖胃)", "🥬 大白菜 (百菜之王)"],
        "tip": "下雪路滑，出门一定要穿防滑鞋！"
    }
}

# --- B. 孕周百科库 (加入了您要求的细节) ---
# 您可以照着这个格式，把 1-40 周慢慢补全
PREGNANCY_DB = {
    # 早期示例
    4: {
        "fruit": "🌰 罂粟籽",
        "size": "像一颗小种子，正在努力着床",
        "development": "👀 **五官**：还是个小胚胎，看不出五官呢。\n💓 **心脏**：血管开始搏动了！",
        "sex_life": "🚫 **关于同房**：\n**达咩！** 胚胎刚着床，非常不稳定，这时候千万不能同房，要像大熊猫一样保护自己。",
        "focus": "补充叶酸，预防神经管畸形。",
        "recipes": [{"name": "菠菜猪肝汤", "tag": "补叶酸"}, {"name": "全麦面包", "tag": "缓解孕吐"}]
    },
    # 您要求的 14周
    14: {
        "fruit": "🍋 柠檬",
        "size": "像个拳头大，重约43克",
        "development": "👀 **眼睛**：已经从侧面移到正脸了，虽然闭着，但能感受到光。\n👃 **鼻子**：小鼻梁挺起来了，长得像爸爸还是妈妈呢？\n✋ **动作**：会皱眉、做鬼脸，还会吸吮大拇指哦！",
        "sex_life": "✅ **关于同房**：\n**可以适度。** 进入孕中期，胎盘稳固了。如果产检正常（无前置胎盘、出血），可以进行温柔的夫妻生活，这有助于增进感情哦。",
        "focus": "胃口恢复，开始长胎，补充优质蛋白。",
        "recipes": [{"name": "彩椒牛柳", "tag": "补铁补维C"}, {"name": "核桃杂粮粥", "tag": "补脑防便秘"}, {"name": "清蒸鲈鱼", "tag": "长胎不长肉"}]
    },
    # 中期示例
    28: {
        "fruit": "🍆 茄子",
        "size": "像个大茄子，重约1000克",
        "development": "👀 **眼睛**：宝宝能睁开眼睛啦！还能转动眼珠到处看。\n🧠 **大脑**：大脑皮层出现皱褶，此时是补脑黄金期。",
        "sex_life": "⚠️ **关于同房**：\n**谨慎小心。** 肚子变大了，如果要同房，尽量避免压迫腹部的姿势，且频率要低。",
        "focus": "预防孕晚期水肿，控制糖分。",
        "recipes": [{"name": "冬瓜排骨汤", "tag": "去水肿"}, {"name": "芹菜炒百合", "tag": "控血压"}]
    },
    # 晚期示例
    38: {
        "fruit": "🍉 西瓜",
        "size": "像个大西瓜，随时准备见面",
        "development": "💇 **头发**：头发已经长得很好了，甚至有3厘米长。\n💪 **状态**：他在练习呼吸和吞咽，蓄势待发！",
        "sex_life": "🚫 **关于同房**：\n**严禁！** 随时可能临产，同房容易引起宫缩破水，容易感染，爸爸要忍一忍哦！",
        "focus": "储备分娩能量，助产。",
        "recipes": [{"name": "莲藕排骨汤", "tag": "清热凉血"}, {"name": "茉莉花苞茶", "tag": "软化宫颈"}]
    },
    # 默认兜底
    "default": {
        "fruit": "🎁 盲盒",
        "size": "正在健康成长",
        "development": "宝宝每天都在变化，今天是充满希望的一天！",
        "sex_life": "⚠️ **关于同房**：请咨询医生意见，遵医嘱。",
        "focus": "营养均衡，少食多餐。",
        "recipes": [{"name": "时蔬瘦肉粥", "tag": "易消化"}, {"name": "白灼虾", "tag": "补蛋白"}]
    }
}

# ================= 3. 工具函数 =================

USER_FILE = "user_profile.json"

def get_current_season_info():
    """根据当前月份返回季节信息"""
    month = datetime.now().month
    for key, data in SEASONAL_DB.items():
        if month in data["months"]:
            return data
    return SEASONAL_DB["spring"] # 默认

def save_user_date(date_obj):
    with open(USER_FILE, "w") as f:
        json.dump({"lmp": date_obj.strftime("%Y-%m-%d")}, f)

def load_user_date():
    if not os.path.exists(USER_FILE): return None
    try:
        with open(USER_FILE, "r") as f:
            data = json.load(f)
            return datetime.strptime(data["lmp"], "%Y-%m-%d").date()
    except: return None

def calculate_weeks(lmp_date):
    today = datetime.now().date()
    delta = today - lmp_date
    weeks = delta.days // 7
    days = delta.days % 7
    return weeks, days

# ================= 4. 主程序逻辑 =================

user_lmp = load_user_date()
season_data = get_current_season_info() # 获取当季信息

# --- 注册页 ---
if not user_lmp:
    st.markdown("<br>", unsafe_allow_html=True)
    c1, c2, c3 = st.columns([1, 2, 1])
    with c2:
        st.image("https://cdn-icons-png.flaticon.com/512/3064/3064032.png", width=100)
        st.title("好孕签 · 开启旅程")
        st.info("为了提供【周数+季节】的双重推荐，请设置您的末次月经日期。")
        d = st.date_input("📅 末次月经第一天", max_value=datetime.today())
        if st.button("🚀 生成我的专属指南", type="primary"):
            save_user_date(d)
            st.rerun()

# --- 专属首页 ---
else:
    weeks, days = calculate_weeks(user_lmp)
    # 匹配周数数据，没有就用默认
    w_data = PREGNANCY_DB.get(weeks, PREGNANCY_DB["default"])
    if weeks > 40: weeks = 40

    # 侧边栏
    with st.sidebar:
        st.image("https://cdn-icons-png.flaticon.com/512/3064/3064032.png", width=60)
        st.markdown(f"### 🤰 孕 {weeks}周+{days}天")
        st.markdown(f"当前季节：**{season_data['name']}**")
        st.progress(min(weeks/40, 1.0))
        if st.button("🔄 修改日期"):
            os.remove(USER_FILE)
            st.rerun()
        st.markdown("---")
        menu = st.radio("功能", ["🏠 今日指南", "🧧 每日一签", "🍱 怎么吃(合集)"])

    # 主内容
    if menu == "🏠 今日指南":
        st.markdown(f"## 👋 早上好，孕 {weeks} 周的准妈妈！")
        
        # 1. 核心看板 (宝宝状态)
        c1, c2 = st.columns([2, 1])
        with c1:
            st.markdown(f"""
            <div class="dashboard-card" style="border-left: 5px solid #ff2442;">
                <div style="display:flex; align-items:center;">
                    <div style="font-size: 50px; margin-right:15px;">{w_data['fruit'].split(' ')[0]}</div>
                    <div>
                        <h3 style="margin:0;">宝宝像 {w_data['fruit'].split(' ')[1]} 啦</h3>
                        <p style="color:#666; font-size:14px;">{w_data['size']}</p>
                    </div>
                </div>
                <hr style="border-top:1px dashed #eee;">
                <p><strong>👶 发育进度：</strong></p>
                <p style="line-height:1.6; color:#444; font-size:15px;">{w_data['development'].replace('\n', '<br>')}</p>
            </div>
            """, unsafe_allow_html=True)
            
        # 2. 季节天气看板 (新功能)
        with c2:
            st.markdown(f"""
            <div class="season-card">
                <h4 style="margin:0; color:#006064;">📅 当季提醒 ({datetime.now().month}月)</h4>
                <p style="font-size:13px; margin-top:5px;">{season_data['desc']}</p>
                <p style="font-size:13px; background:#fff; padding:5px; border-radius:5px; margin-top:5px;">
                    💡 <strong>小贴士：</strong>{season_data['tip']}
                </p>
            </div>
            """, unsafe_allow_html=True)
            
            # 同房建议卡片
            sex_color = "#e8f5e9" if "可以" in w_data['sex_life'] else "#ffebee"
            sex_icon = "🟢" if "可以" in w_data['sex_life'] else "🔴"
            with st.expander(f"{sex_icon} 羞羞的事 (同房建议)", expanded=True):
                st.write(w_data['sex_life'])

        # 3. 智能饮食推荐 (合并了周数+季节)
        st.markdown("### 🥗 今日专属食谱推荐")
        st.caption(f"根据 **孕{weeks}周需求** + **{season_data['name']}时令** 为您生成")
        
        col1, col2 = st.columns(2)
        
        with col1:
            st.markdown("#### 🤰 针对孕周 (补重点)")
            st.info(f"本周重点：{w_data['focus']}")
            for r in w_data['recipes']:
                st.markdown(f"""
                <div class="recipe-box">
                    <strong>🥘 {r['name']}</strong>
                    <span class="tag tag-week">{r['tag']}</span>
                </div>
                """, unsafe_allow_html=True)
                
        with col2:
            st.markdown(f"#### 📅 针对季节 (吃时令)")
            st.success(f"当季推荐：{', '.join(season_data['eat'][:2])}")
            # 动态生成几个季节性菜谱
            season_recipes = [f"清炒{season_data['eat'][2].split(' ')[0]}", f"{season_data['eat'][0].split(' ')[0]}沙拉"]
            for r in season_recipes:
                st.markdown(f"""
                <div class="recipe-box">
                    <strong>🥗 {r}</strong>
                    <span class="tag tag-season">应季鲜食</span>
                </div>
                """, unsafe_allow_html=True)

    # 保留每日一签功能
    elif menu == "🧧 每日一签":
        st.header("🧧 每日灵签")
        if st.button("📿 点击求签"):
            lucks = ["上上签：宜开心", "中平签：宜吃肉", "大吉签：宝宝很乖"]
            st.success(random.choice(lucks))
            st.balloons()

# 底部
st.markdown("<div class='footer'>好孕签 App | 科学孕育 · 仅供参考 · 身体不适请及时就医</div>", unsafe_allow_html=True)# haoyun
