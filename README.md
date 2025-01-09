# SWAP
Sample code for preview prototype of SWAP
from fastapi import FastAPI, HTTPException, Depends
from fastapi.security import OAuth2PasswordBearer, OAuth2PasswordRequestForm
from pydantic import BaseModel
from typing import List
from passlib.context import CryptContext
import uvicorn

# Initialize the app
app = FastAPI()

# Security setup
pwd_context = CryptContext(schemes=["bcrypt"], deprecated="auto")
oauth2_scheme = OAuth2PasswordBearer(tokenUrl="/login")

# Mock database
users_db = {
    "admin@example.com": {
        "username": "admin",
        "full_name": "Admin User",
        "email": "admin@example.com",
        "hashed_password": pwd_context.hash("admin123"),
        "is_admin": True,
        "skills": []
    },
    "user@example.com": {
        "username": "user",
        "full_name": "Regular User",
        "email": "user@example.com",
        "hashed_password": pwd_context.hash("user123"),
        "is_admin": False,
        "skills": []
    }
}

# Models
class User(BaseModel):
    username: str
    full_name: str
    email: str
    is_admin: bool
    skills: List[str]

class Skill(BaseModel):
    title: str
    description: str

# Utility functions
def get_user(email: str):
    user = users_db.get(email)
    if user:
        return user
    return None

def verify_password(plain_password, hashed_password):
    return pwd_context.verify(plain_password, hashed_password)

def authenticate_user(email: str, password: str):
    user = get_user(email)
    if not user or not verify_password(password, user["hashed_password"]):
        return None
    return user

# Routes
@app.post("/login")
async def login(form_data: OAuth2PasswordRequestForm = Depends()):
    user = authenticate_user(form_data.username, form_data.password)
    if not user:
        raise HTTPException(status_code=401, detail="Invalid credentials")
    return {"access_token": user["email"], "token_type": "bearer"}

@app.get("/users/me", response_model=User)
async def get_current_user(token: str = Depends(oauth2_scheme)):
    user = get_user(token)
    if not user:
        raise HTTPException(status_code=401, detail="Invalid token")
    return User(
        username=user["username"],
        full_name=user["full_name"],
        email=user["email"],
        is_admin=user["is_admin"],
        skills=user["skills"]
    )

@app.post("/skills")
async def add_skill(skill: Skill, token: str = Depends(oauth2_scheme)):
    user = get_user(token)
    if not user:
        raise HTTPException(status_code=401, detail="Invalid token")

    user["skills"].append(skill.title)
    return {"message": f"Skill '{skill.title}' added successfully!", "skills": user["skills"]}

@app.get("/skills", response_model=List[str])
async def get_skills(token: str = Depends(oauth2_scheme)):
    user = get_user(token)
    if not user:
        raise HTTPException(status_code=401, detail="Invalid token")

    return user["skills"]

@app.delete("/skills/{skill_title}")
async def delete_skill(skill_title: str, token: str = Depends(oauth2_scheme)):
    user = get_user(token)
    if not user:
        raise HTTPException(status_code=401, detail="Invalid token")

    if skill_title not in user["skills"]:
        raise HTTPException(status_code=404, detail="Skill not found")

    user["skills"].remove(skill_title)
    return {"message": f"Skill '{skill_title}' removed successfully!", "skills": user["skills"]}

# SaaS Features
@app.get("/admin/users", response_model=List[User])
async def get_all_users(token: str = Depends(oauth2_scheme)):
    user = get_user(token)
    if not user or not user["is_admin"]:
        raise HTTPException(status_code=403, detail="Admin access required")

    return [
        User(
            username=u["username"],
            full_name=u["full_name"],
            email=u["email"],
            is_admin=u["is_admin"],
            skills=u["skills"]
        ) for u in users_db.values()
    ]

if __name__ == "__main__":
    uvicorn.run(app, host="0.0.0.0", port=8000)
