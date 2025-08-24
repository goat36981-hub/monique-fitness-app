# monique-fitness-app
An fitness coach who loves helping people reach there goals
# ====================== REPO START ======================

# filepath: requirements.txt
streamlit>=1.31
pillow>=10.3
pytest>=7.4

# filepath: README.md
# Monique Frazier — AI Fitness Model (Instagram Pack)
Web app + IG carousel generator for Monique Frazier (aka @sexy_mo23).

## Local dev
pip install -r requirements.txt
streamlit run app.py

## Tests
pytest -q

## Deploy (Streamlit Cloud)
- Repo must contain: app.py, ig_fitness_monique_toolkit.py, requirements.txt.
- Deploy settings:
  - Repository: https://github.com/<your-user>/<repo>
  - Branch: main
  - Main file path: app.py
  - App URL: monique-fitness (or similar; avoid "sexy" due to filter)

## IG usage
Generate a plan → Download ZIP → upload PNGs as a carousel → paste caption.txt.

# filepath: ig_fitness_monique_toolkit.py
from __future__ import annotations
import csv, json, math, os, io, zipfile
from dataclasses import dataclass, asdict
from typing import Dict, List, Optional, Tuple
from PIL import Image, ImageDraw, ImageFont

@dataclass
class UserProfile:
    name: str; handle: str; age: int; sex: str
    height_cm: float; weight_kg: float
    city: str; origin: str; years_experience: int
    primary_goal: str; fitness_level: str; equipment: str
    dietary_pref: str; restrictions: List[str]

@dataclass
class WorkoutDay:
    day: str; focus: str; session: List[str]; est_duration_min: int; notes: str=""

@dataclass
class Program:
    profile: UserProfile; bmi: float; bmr: int; tdee: int
    calories_target: int; macros: Dict[str,int]
    goals_statement: str; health_benefits: List[str]
    weekly_plan: List[WorkoutDay]; nutrition_guidance: List[str]
    sample_meals: Dict[str, List[str]]; recovery_habits: List[str]
    progression_notes: List[str]

def calc_bmi(w: float, h_cm: float) -> float:
    if h_cm<=0: raise ValueError("height_cm must be > 0")
    return round(w/((h_cm/100)**2), 1)

def calc_bmr_msj(sex: str, w: float, h_cm: float, age: int) -> int:
    b = 10*w + 6.25*h_cm - 5*age + (5 if sex.lower()!="female" else -161)
    return int(round(b))

def activity_factor(level: str) -> float:
    return {"beginner":1.4,"intermediate":1.55,"advanced":1.7}.get(level.lower(),1.5)

def target_calories(goal: str, tdee: int) -> int:
    g=goal.lower()
    if g in {"weight loss","cut"}: return int(round(tdee*0.8))
    if g in {"muscle gain","bulk"}: return int(round(tdee*1.1))
    if g=="endurance": return int(round(tdee))
    return int(round(tdee*0.95))

def macro_split(goal: str, calories: int, w: float) -> Dict[str,int]:
    protein_g=(1.8 if goal.lower()=="muscle gain" else 1.7)*w
    fat_g=max(0.8*w,45)
    remaining=max(calories-int(round(protein_g*4+fat_g*9)), int(0.25*calories))
    carbs_g=remaining/4
    return {"protein_g":int(round(protein_g)),"carbs_g":int(round(carbs_g)),"fat_g":int(round(fat_g))}

GOALS_MAP={
    "weight loss":"Reduce body fat while maintaining lean muscle and metabolic health.",
    "muscle gain":"Increase lean mass, strength, and shape with progressive overload.",
    "endurance":"Improve cardiovascular capacity, pacing, and recovery under fatigue.",
    "recomp":"Build muscle and reduce fat via smart nutrition and training.",
    "general fitness":"Enhance strength, mobility, stamina, and daily energy.",
}
BENEFITS_MAP={
    "weight loss":["Improved insulin sensitivity","Lower blood pressure","Reduced joint stress","Better sleep quality"],
    "muscle gain":["Higher resting metabolism","Bone density support","Improved posture","Functional strength"],
    "endurance":["Enhanced VO₂ efficiency","Lower resting HR","Stress resilience","Improved circulation"],
    "recomp":["Stable energy","Body shape refinement","Better nutrient partitioning","Sustainable habits"],
    "general fitness":["Mood elevation","Injury risk reduction","Mobility & balance","Longevity markers"],
}
MEALS_BANK={
    "none":{
        "breakfast":["Greek yogurt + berries + honey + granola","Veggie omelet + whole‑grain toast","Overnight oats + chia + banana"],
        "lunch":["Grilled chicken bowl (rice, beans, pico, avocado)","Turkey wrap + greens","Seared salmon salad + quinoa"],
        "dinner":["Lean steak + sweet potato + broccoli","Shrimp stir‑fry + jasmine rice","Tofu curry + brown rice"],
        "snacks":["Protein shake","Apple + peanut butter","Cottage cheese + pineapple"],
    },
    "vegetarian":{
        "breakfast":["Oats + protein + almond butter","Scrambled eggs + veggies","Greek yogurt parfait"],
        "lunch":["Halloumi wrap + hummus","Lentil bowl + roasted veg","Caprese + farro"],
        "dinner":["Paneer tikka + basmati","Tofu stir‑fry + noodles","Bean chili + corn bread"],
        "snacks":["Trail mix","Protein shake","Carrots + hummus"],
    },
    "vegan":{
        "breakfast":["Tofu scramble + toast","Overnight oats + pea protein","Smoothie bowl"],
        "lunch":["Chickpea quinoa bowl","Tempeh tacos","Vegan sushi + edamame"],
        "dinner":["Lentil bolognese + pasta","Coconut curry tofu + rice","Seitan fajitas + peppers"],
        "snacks":["Edamame","Almonds","Fruit + soy yogurt"],
    },
}

def build_weekly_plan(level: str, equip: str, goal: str) -> List[WorkoutDay]:
    def s(lo,hi): return f"3–4 sets × {lo}–{hi} reps"
    gym_push=[f"Barbell bench ({s(6,10)})",f"DB shoulder press ({s(8,12)})",f"Incline DB press ({s(8,12)})",f"Triceps pushdown ({s(10,15)})","Incline walk 10–12 min"]
    gym_pull=[f"RDL ({s(5,8)})",f"Lat pulldown/pull‑ups ({s(6,10)})",f"Seated row ({s(8,12)})",f"Face pulls ({s(12,15)})","Plank 3×45–60s"]
    gym_legs=[f"Back/front squat ({s(5,8)})",f"Romanian deadlift ({s(6,10)})",f"Leg press or split squats ({s(8,12)})",f"Hamstring curls ({s(10,15)})","Calf raises 4×12–15"]
    home_strength=["Push‑ups 4×AMRAP","Bulgarian split squats 4×10–12/leg","Hip thrusts 4×12–15","One‑arm row 4×10–12/side","Hollow hold 3×30–45s"]
    conditioning=["Intervals: 8×(1' hard / 1' easy)","Steady cardio 30–40 min Z2"]
    mobility=["Warm‑up 10 min","Mobility 15 min (hips/shoulders/ankles)","Breathing 5 min"]
    equip=equip.lower()
    if equip=="gym":
        days=[WorkoutDay("Mon","Lower strength",gym_legs,60,"Leave 1–2 RIR."),
              WorkoutDay("Tue","Conditioning + core",[conditioning[0],"Core circuit 10–12 min"],40),
              WorkoutDay("Wed","Push (chest/shoulders)",gym_push,60),
              WorkoutDay("Thu","Mobility + steady cardio",[mobility[1],conditioning[1]],45),
              WorkoutDay("Fri","Pull (back/hamstrings)",gym_pull,60),
              WorkoutDay("Sat","Glutes/isolations + class",["Cable kickbacks 4×12–15","Leg extensions 3×12–15","Lateral raises 3×12–15","Pilates/Yoga 30 min"],55),
              WorkoutDay("Sun","Rest / walk",["10k steps","Stretch 10 min"],20,"Reflect & plan.")]
    elif equip=="home":
        days=[WorkoutDay("Mon","Full‑body (home)",home_strength,45),
              WorkoutDay("Tue","Intervals",[conditioning[0]],30),
              WorkoutDay("Wed","Glutes/core",["Single‑leg RDL 4×10/leg","Banded walks 4×15","Side plank 3×45s"],40),
              WorkoutDay("Thu","Mobility",mobility,30),
              WorkoutDay("Fri","Full‑body (home)",home_strength,45),
              WorkoutDay("Sat","Steady cardio",[conditioning[1]],35),
              WorkoutDay("Sun","Rest / steps",["8–10k steps","Gentle stretch"],20)]
    else:
        days=[WorkoutDay("Mon","Lower (gym)",gym_legs,60),
              WorkoutDay("Tue","Home intervals",[conditioning[0]],30),
              WorkoutDay("Wed","Upper push (gym)",gym_push,60),
              WorkoutDay("Thu","Mobility",mobility,30),
              WorkoutDay("Fri","Upper pull (gym)",gym_pull,60),
              WorkoutDay("Sat","Steady cardio + core",[conditioning[1],"Core circuit 10–12 min"],45),
              WorkoutDay("Sun","Rest",["10k steps","Light yoga"],20)]
    if goal.lower()=="endurance":
        for d in days:
            if "cardio" in d.focus.lower() or "interval" in d.focus.lower():
                d.est_duration_min=max(d.est_duration_min,45)
    return days

def generate_program(profile: UserProfile) -> Program:
    bmi=calc_bmi(profile.weight_kg, profile.height_cm)
    bmr=calc_bmr_msj(profile.sex, profile.weight_kg, profile.height_cm, profile.age)
    tdee=int(round(bmr*activity_factor(profile.fitness_level)))
    calories=target_calories(profile.primary_goal, tdee)
    macros=macro_split(profile.primary_goal, calories, profile.weight_kg)
    weekly_plan=build_weekly_plan(profile.fitness_level, profile.equipment, profile.primary_goal)
    diet_key=profile.dietary_pref if profile.dietary_pref in MEALS_BANK else "none"
    return Program(
        profile=profile, bmi=bmi, bmr=bmr, tdee=tdee, calories_target=calories, macros=macros,
        goals_statement=GOALS_MAP.get(profile.primary_goal, GOALS_MAP["general fitness"]),
        health_benefits=BENEFITS_MAP.get(profile.primary_goal, BENEFITS_MAP["general fitness"]),
        weekly_plan=weekly_plan,
        nutrition_guidance=[
            f"Calories target: ~{calories} kcal (from TDEE {tdee}).",
            f"Macros: {macros['protein_g']}g P / {macros['carbs_g']}g C / {macros['fat_g']}g F.",
            "Protein every meal; veggies 2×+ daily; hydrate 30–35 ml/kg.",
            "Pre‑workout: carbs+protein; Post: 25–35g protein + carbs.",
            "80/20 rule: mostly whole foods.",
        ] + ([f"Restrictions: {', '.join(profile.restrictions)}."] if profile.restrictions else []),
        sample_meals=MEALS_BANK[diet_key],
        recovery_habits=[
            "Sleep 7–9h; consistent schedule.","Rest day walk + mobility 15 min.","Daily steps 8–12k.",
            "Deload every 6–8 weeks if fatigue rises.","Electrolytes on hot/humid days (Baton Rouge).",
        ],
        progression_notes=[
            "Progress when last set reaches top reps with ~2 RIR.",
            "Add 1–2 sets per muscle across 4–6 weeks, then deload.",
            "Adjust calories ±100–150 if weight trend stalls 2+ weeks.",
        ],
    )

def default_monique(h_cm: float=167.0, w_kg: float=62.0, goal: str="general fitness",
                    level: str="intermediate", equip: str="mixed", diet: str="none")->UserProfile:
    return UserProfile("Monique Frazier","sexy_mo23",23,"female",h_cm,w_kg,"Baton Rouge","Washington, DC",
                       5,goal,level,equip,diet,[])

class ProgressTracker:
    def __init__(self, program: Program, store: str="progress.csv"):
        self.program=program; self.store=store; self.rows: List[Dict[str,str]]=[]
        if os.path.exists(store): self._load()
    def add_checkin(self, date: str, w: float, waist_cm: float, energy: int, compliance: float)->None:
        self.rows.append({"date":date,"weight_kg":f"{w:.2f}","waist_cm":f"{waist_cm:.1f}","energy":str(int(energy)),"compliance":f"{compliance:.2f}"}); self._save()
    def trend_weekly_change(self)->Optional[float]:
        if len(self.rows)<2: return None
        try: return round(float(self.rows[-1]["weight_kg"])-float(self.rows[-2]["weight_kg"]),2)
        except Exception: return None
    def auto_adjust_calories(self)->Tuple[int,str]:
        ch=self.trend_weekly_change(); cals=self.program.calories_target
        if ch is None: return cals,"Need ≥2 check-ins."
        goal=self.program.profile.primary_goal.lower(); comp=float(self.rows[-1]["compliance"])
        if comp<0.8: return cals,"No change (low compliance)."
        if goal in {"weight loss","recomp"}:
            if ch>-0.15: return max(cals-120,1200),"Deficit increased by ~120 kcal."
            if ch<-1.0: return cals+120,"Deficit reduced by ~120 kcal."
        if goal=="muscle gain":
            if ch<0.1: return cals+120,"Surplus increased by ~120 kcal."
            if ch>0.6: return cals-120,"Surplus reduced by ~120 kcal."
        return cals,"No change."
    def _save(self):
        with open(self.store,"w",newline="",encoding="utf-8") as f:
            w=csv.DictWriter(f,fieldnames=["date","weight_kg","waist_cm","energy","compliance"])
            w.writeheader(); w.writerows(self.rows)
    def _load(self):
        with open(self.store,"r",encoding="utf-8") as f: self.rows=list(csv.DictReader(f))

def _get_font(size:int):
    try: return ImageFont.truetype("DejaVuSans.ttf", size)
    except Exception: return ImageFont.load_default()

def _wrap(draw, text, font, max_w)->List[str]:
    words=text.split(); lines=[]; cur=""
    for w in words:
        t=(cur+" "+w).strip()
        if draw.textlength(t,font=font)<=max_w: cur=t
        else: lines.append(cur); cur=w
    if cur: lines.append(cur); return lines

def render_slide(title: str, bullets: List[str]) -> Image.Image:
    W,H=1080,1350; img=Image.new("RGB",(W,H),(245,245,245)); d=ImageDraw.Draw(img)
    title_font=_get_font(72); body_font=_get_font(44); small_font=_get_font(34)
    y=80; d.text((80,y),title,font=title_font,fill=(0,0,0)); y+=120; max_w=W-160
    for b in bullets:
        for i,line in enumerate(_wrap(d,b,body_font,max_w)):
            d.text((120 if i==0 else 140,y),("• " if i==0 else "  ")+line,font=body_font,fill=(15,15,15)); y+=60
        y+=10
        if y>H-220: break
    d.text((80,H-120),"Monique Frazier · @sexy_mo23",font=small_font,fill=(60,60,60))
    return img

def make_instagram_images(program: Program) -> List[Tuple[str, Image.Image]]:
    slides=[]
    bullets=[program.goals_statement,f"BMI {program.bmi} | BMR {program.bmr} | TDEE ~{program.tdee}",
             f"Target ~{program.calories_target} kcal  •  Macros {program.macros['protein_g']}P/{program.macros['carbs_g']}C/{program.macros['fat_g']}F",
             "Swipe ➜ for plan, meals, recovery."]
    slides.append(("01_title.png", render_slide("My Week — Monique", bullets)))
    slides.append(("02_benefits.png", render_slide("Why this works", program.health_benefits[:8])))
    days=program.weekly_plan
    for i in range(0,len(days),3):
        chunk=days[i:i+3]; b=[]
        for d in chunk:
            b.append(f"{d.day}: {d.focus} (~{d.est_duration_min}m)")
            b += [f"  - {x}" for x in d.session[:4]]
        slides.append((f"{3+i//3:02d}_plan.png", render_slide("Weekly Training", b)))
    b=program.nutrition_guidance[:8]+[f"Meals: {', '.join(program.sample_meals['breakfast'][:2])}…"]
    slides.append(("05_nutrition.png", render_slide("Nutrition & Macros", b)))
    slides.append(("06_recovery.png", render_slide("Recovery & Habits", program.recovery_habits[:8])))
    slides.append(("07_cta.png", render_slide("Want yours?", ["Comment “PROGRAM”","I’ll DM you a personalized plan ♥","Save & share this carousel"])))
    return slides

def build_hashtags(profile: UserProfile, goal: str) -> List[str]:
    base=["fitness","fitmodel","workout","nutrition","health","training","batonrouge","dc","gymlife","fitspo","gluteworkout","cardio","strength"]
    goal_tags={"weight loss":["fatloss","caloriedeficit"],"muscle gain":["musclegain","progressiveoverload"],"endurance":["endurance","running"],"recomp":["bodyrecomp"],"general fitness":["wellness","mobility"]}
    handle_tags=[profile.handle,"moniquefrazier"]; return ["#"+t for t in (handle_tags+goal_tags.get(goal,[])+base)]

def build_caption(p: Program) -> str:
    prof=p.profile
    lines=[f"{prof.name} ({prof.handle}) — DC ➜ {prof.city} | 23 | {prof.years_experience} yrs training 💪",
           f"Goal: {prof.primary_goal.capitalize()}",
           f"Calories ~{p.calories_target} • Macros {p.macros['protein_g']}P/{p.macros['carbs_g']}C/{p.macros['fat_g']}F",
           f"Benefits: {', '.join(p.health_benefits[:3])}…","Swipe ➜ for weekly plan, meals, and recovery tips.",
           "Comment 'PROGRAM' for a DM with your personalized version. 🔥"]
    tags=" ".join(build_hashtags(prof, prof.primary_goal)[:18]); return "\n".join(lines)+"\n\n"+tags

def package_ig_zip(program: Program) -> bytes:
    slides=make_instagram_images(program)
    caption=build_caption(program)
    buf=io.BytesIO()
    with zipfile.ZipFile(buf,"w",zipfile.ZIP_DEFLATED) as zf:
        for name,img in slides:
            png=io.BytesIO(); img.save(png,format="PNG"); zf.writestr(name,png.getvalue())
        zf.writestr("caption.txt", caption.encode("utf-8"))
        zf.writestr("program.json", json.dumps(asdict(program), indent=2).encode("utf-8"))
    return buf.getvalue()

# filepath: app.py
import streamlit as st
from ig_fitness_monique_toolkit import (
    default_monique, generate_program, build_caption, package_ig_zip
)

st.set_page_config(page_title="Monique Frazier Fitness AI", page_icon="💪", layout="centered")
st.title("💪 Monique Frazier — AI Fitness Model")
st.caption("aka @sexy_mo23 · DC ➜ Baton Rouge · 5 years experience")

with st.form("inputs"):
    col1,col2=st.columns(2)
    with col1:
        height=st.number_input("Height (cm)",140.0,200.0,167.0,0.5)
        weight=st.number_input("Weight (kg)",40.0,140.0,62.0,0.1)
        level=st.selectbox("Fitness level",["beginner","intermediate","advanced"],index=1)
    with col2:
        goal=st.selectbox("Primary goal",["general fitness","weight loss","muscle gain","recomp","endurance"])
        equip=st.selectbox("Equipment",["home","gym","mixed"],index=2)
        diet=st.selectbox("Dietary preference",["none","vegetarian","vegan"],index=0)
    restrictions=st.text_input("Restrictions (comma‑separated)", "")
    submitted=st.form_submit_button("Generate Program", use_container_width=True)

if submitted:
    profile=default_monique(height,weight,goal,level,equip,diet)
    if restrictions.strip(): profile.restrictions=[r.strip() for r in restrictions.split(",") if r.strip()]
    program=generate_program(profile)

    st.subheader("Overview")
    st.write({"BMI":program.bmi,"BMR":program.bmr,"TDEE":program.tdee,
              "Calories target":program.calories_target,"Macros":program.macros})

    st.subheader("Goals & Benefits")
    st.write(program.goals_statement); st.write(program.health_benefits)

    st.subheader("Weekly Plan")
    for d in program.weekly_plan:
        with st.expander(f"{d.day} — {d.focus} (~{d.est_duration_min} min)"):
            for item in d.session: st.markdown(f"- {item}")
            if d.notes: st.caption(d.notes)

    st.subheader("Nutrition & Recovery")
    st.write(program.nutrition_guidance); st.write(program.sample_meals); st.write(program.recovery_habits)

    st.subheader("Instagram Caption")
    st.code(build_caption(program), language="markdown")

    st.subheader("Instagram Carousel")
    zip_bytes=package_ig_zip(program)
    st.download_button("Download IG Slides (.zip)", data=zip_bytes,
                       file_name="monique_ig_pack.zip", mime="application/zip")

# filepath: tests/test_toolkit.py
from ig_fitness_monique_toolkit import (
    calc_bmi, calc_bmr_msj, activity_factor, target_calories, macro_split,
    default_monique, generate_program, ProgressTracker
)

def test_bmi(): assert calc_bmi(62,167)==22.2
def test_bmr_female(): assert 1350 <= calc_bmr_msj("female",62,167,23) <= 1500
def test_activity_factor(): assert activity_factor("advanced")>activity_factor("beginner")
def test_macros_total_close():
    prog=generate_program(default_monique())
    m=prog.macros; total=m["protein_g"]*4+m["carbs_g"]*4+m["fat_g"]*9
    assert abs(total-prog.calories_target)<400
def test_adjust_weight_loss(tmp_path):
    prog=generate_program(default_monique(goal="weight loss"))
    from ig_fitness_monique_toolkit import ProgressTracker
    tr=ProgressTracker(prog, store=str(tmp_path/"p.csv"))
    tr.add_checkin("2025-08-18", prog.profile.weight_kg, 72, 7, 0.95)
    tr.add_checkin("2025-08-25", prog.profile.weight_kg-0.05, 71.8, 7, 0.95)
    new_cals,_=tr.auto_adjust_calories()
    assert new_cals<=prog.calories_target

# ======================= REPO END =======================