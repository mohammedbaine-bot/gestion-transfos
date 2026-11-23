import streamlit as st
import pandas as pd
from datetime import datetime
from streamlit_gsheets import GSheetsConnection

st.set_page_config(page_title="Gestion des Postes", layout="wide")

# الاتصال بـ Google Sheets
conn = st.connection("gsheets", type=GSheetsConnection)

def load_data():
    try:
        df_base = conn.read(worksheet="Base_Actuelle", ttl=0).dropna(how="all").fillna("")
        df_archive = conn.read(worksheet="Archive_Mouvements", ttl=0).dropna(how="all").fillna("")
        return df_base, df_archive
    except Exception as e:
        st.error(f"⚠️ مشكل في الاتصال بـ Google Sheets: {e}")
        return None, None

def save_data(df_base, df_archive):
    conn.update(worksheet="Base_Actuelle", data=df_base)
    conn.update(worksheet="Archive_Mouvements", data=df_archive)
    st.cache_data.clear()

def extraire_chiffre(valeur):
    try:
        if pd.isna(valeur) or valeur == "": return 0
        return float(''.join(filter(lambda x: x.isdigit() or x == '.', str(valeur))))
    except: return 0

def calculer_charge(data_dict):
    try:
        i1, i2, i3 = float(data_dict.get("I1  A") or 0), float(data_dict.get("I2 A") or 0), float(data_dict.get("I3 A") or 0)
        pui_nominale = extraire_chiffre(str(data_dict.get("PUI", 0)))
        if i1 > 0 or i2 > 0 or i3 > 0:
            i_moyen = (i1 + i2 + i3) / 3
            data_dict["I moyenne"] = f"{i_moyen:.2f} A"
            p_app = (400 * i_moyen * 1.732) / 1000
            data_dict["PUISSSANCE APPELE"] = f"{p_app:.2f} kVA"
            data_dict["LA CHARGE TRANSFO EN %"] = f"{(p_app / pui_nominale) * 100:.2f} %" if pui_nominale > 0 else "خطأ: PUI غائبة"
            return True, data_dict["I moyenne"], data_dict["PUISSSANCE APPELE"], data_dict["LA CHARGE TRANSFO EN %"]
    except ValueError: pass 
    return False, "", "", ""

st.title("⚡ منصة تدبير المحولات (En Ligne)")
df_base, df_archive = load_data()

if df_base is not None:
    action = st.radio("العملية المراد إنجازها:", ["📊 تحديث معطيات POSTE", "🔄 حركة وتغيير محول", "➕ إضافة جديد"])
    st.markdown("---")

    if action == "📊 تحديث معطيات POSTE":
        postes_list = [p for p in df_base['NOM POSTE'].unique() if p != "STOCKS EXPLOITATION" and p != ""]
        selected_poste = st.selectbox("اختر المركز:", ["-- اختر --"] + postes_list)
        if selected_poste != "-- اختر --":
            idx = df_base.index[df_base['NOM POSTE'] == selected_poste].tolist()[0]
            poste_data = df_base.iloc[idx]
            with st.form("update_form"):
                new_data = {}; cols = st.columns(3)
                auto, bloq = ["I moyenne", "PUISSSANCE APPELE", "LA CHARGE TRANSFO EN %"], ["NOM POSTE", "MARQUE.TR", "N SERIE TR", "PUI", "EMPLACEMENT ACTUEL (POSTE OU STOCK)"]
                for i, col in enumerate(df_base.columns):
                    val = str(poste_data[col])
                    with cols[i % 3]:
                        if col in auto: new_data[col] = st.text_input(f"{col} (أوتوماتيكي)", val, disabled=True)
                        elif col in bloq: new_data[col] = val; st.text_input(f"{col} (مقفول)", val, disabled=True)
                        else: new_data[col] = st.text_input(col, val)
                if st.form_submit_button("💾 تحديث وحفظ"):
                    new_data["PUI"] = poste_data["PUI"] 
                    calculer_charge(new_data)
                    for col in df_base.columns: df_base.at[idx, col] = new_data.get(col, poste_data[col])
                    save_data(df_base, df_archive)
                    st.success("✅ تم التحديث بنجاح في Google Sheets!")

    elif action == "🔄 حركة وتغيير محول":
        transfos = [t for t in df_base['N SERIE TR'].unique() if t != ""]
        selected_transfo = st.selectbox("1. اختر المحول (N SERIE TR):", ["-- اختر --"] + transfos)
        if selected_transfo != "-- اختر --":
            idx = df_base.index[df_base['N SERIE TR'] == selected_transfo].tolist()[0]
            row = df_base.iloc[idx]
            emp_actuel = row['EMPLACEMENT ACTUEL (POSTE OU STOCK)'] if row['EMPLACEMENT ACTUEL (POSTE OU STOCK)'] != "" else row['NOM POSTE']
            st.info(f"📍 المكان الحالي: **{emp_actuel}**")
            dests = ["STOCKS EXPLOITATION"] + [p for p in df_base['NOM POSTE'].unique() if p != "STOCKS EXPLOITATION" and p != ""]
            if emp_actuel in dests: dests.remove(emp_actuel)
            with st.form("mvt_form"):
                nouv_emp = st.selectbox("2. المكان الجديد:", ["-- اختر --"] + dests)
                code_prj = st.text_input("3. كود المشروع:")
                type_mvt = st.selectbox("4. نوع الحركة:", ["تركيب (Pose)", "إزالة (Dépose)", "تغيير (Remplacement)"])
                if st.form_submit_button("🔄 تأكيد الحركة"):
                    if nouv_emp == "-- اختر --": st.error("اختر المكان!")
                    else:
                        archive_row = {"Date Mouvement": datetime.now().strftime("%Y-%m-%d %H:%M:%S"), "Type Mouvement": type_mvt, "CODE IMPUTATION /CODE PROJET": code_prj, "N SERIE TR": selected_transfo, "MARQUE.TR": row.get('MARQUE.TR',''), "PUI": row.get('PUI',''), "Emplacement Precedent": emp_actuel, "Nouvel Emplacement": nouv_emp}
                        df_archive = pd.concat([df_archive, pd.DataFrame([archive_row])], ignore_index=True)
                        cols_tr = ["MARQUE.TR", "AF", "N SERIE TR", "PUI", "D. MS TRANSFO", "Ucc", "Bt"]
                        tr_data = {c: row[c] for c in cols_tr}
                        for c in cols_tr: df_base.at[idx, c] = ""
                        if emp_actuel == "STOCKS EXPLOITATION": df_base = df_base.drop(idx).reset_index(drop=True)
                        if nouv_emp == "STOCKS EXPLOITATION":
                            n_row = {c: "" for c in df_base.columns}; n_row.update(tr_data); n_row["EMPLACEMENT ACTUEL (POSTE OU STOCK)"] = n_row["NOM POSTE"] = "STOCKS EXPLOITATION"
                            df_base = pd.concat([df_base, pd.DataFrame([n_row])], ignore_index=True)
                        else:
                            d_idx = df_base.index[df_base['NOM POSTE'] == nouv_emp].tolist()[0]
                            for c in cols_tr: df_base.at[d_idx, c] = tr_data[c]
                            df_base.at[d_idx, "EMPLACEMENT ACTUEL (POSTE OU STOCK)"] = nouv_emp
                        save_data(df_base, df_archive)
                        st.success(f"✅ تمت الحركة بنجاح إلى {nouv_emp}!")

    elif action == "➕ إضافة جديد":
        type_ajout = st.radio("ماذا تريد أن تنشئ؟", ["🏗️ إنشاء POSTE جديد", "📦 إدخال محول للـ STOCKS"])
        with st.form("ajout_form"):
            new_data = {c: "" for c in df_base.columns}; cols = st.columns(3)
            if type_ajout == "🏗️ إنشاء POSTE جديد":
                for i, col in enumerate(["SECT", "NOM POSTE", "DATEMS", "CEL MOTO", "MARQUE CELLULE", "REG", "TYPE"]):
                    with cols[i % 3]: new_data[col] = st.text_input(col)
                new_data["EMPLACEMENT ACTUEL (POSTE OU STOCK)"] = new_data.get("NOM POSTE", "")
            else:
                for i, col in enumerate(["N SERIE TR", "MARQUE.TR", "PUI", "AF", "D. MS TRANSFO", "Ucc", "Bt", "CODE IMPUTATION /CODE PROJET"]):
                    with cols[i % 3]: new_data[col] = st.text_input(col)
                new_data["NOM POSTE"] = new_data["EMPLACEMENT ACTUEL (POSTE OU STOCK)"] = "STOCKS EXPLOITATION"

            if st.form_submit_button("➕ تأكيد وإضافة"):
                if (type_ajout == "🏗️ إنشاء POSTE جديد" and not new_data.get("NOM POSTE")) or (type_ajout != "🏗️ إنشاء POSTE جديد" and not new_data.get("N SERIE TR")):
                    st.error("❌ أدخل الاسم أو الرقم التسلسلي!")
                else:
                    df_base = pd.concat([df_base, pd.DataFrame([new_data])], ignore_index=True)
                    if type_ajout != "🏗️ إنشاء POSTE جديد":
                        archive_row = {"Date Mouvement": datetime.now().strftime("%Y-%m-%d %H:%M:%S"), "Type Mouvement": "Entrée Stock", "CODE IMPUTATION /CODE PROJET": new_data["CODE IMPUTATION /CODE PROJET"], "N SERIE TR": new_data["N SERIE TR"], "MARQUE.TR": new_data["MARQUE.TR"], "PUI": new_data["PUI"], "Emplacement Precedent": "FOURNISSEUR", "Nouvel Emplacement": "STOCKS EXPLOITATION"}
                        df_archive = pd.concat([df_archive, pd.DataFrame([archive_row])], ignore_index=True)
                    save_data(df_base, df_archive)
                    st.success("✅ تمت الإضافة بنجاح!")
