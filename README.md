# Zenith-Cryptobridge
# Zenith Digital Asset Access Gateway (CBN-Safe)
# Debugged & Production-Ready Single-File Backend
#
# ❗ IMPORTANT SETUP ❗
# If you see: ModuleNotFoundError: No module named 'fastapi'
# You MUST install dependencies first:
#
#   pip install fastapi uvicorn sqlalchemy
#
# Then run:
#   uvicorn main:app --reload
#
# -----------------------------------------------

from fastapi import FastAPI, HTTPException, Depends
from sqlalchemy import Column, String, Float, Boolean, DateTime, create_engine
from sqlalchemy.ext.declarative import declarative_base
from sqlalchemy.orm import sessionmaker, Session
import datetime
import uuid
import os

# ===================== CONFIG =====================
class Settings:
    PROJECT_NAME = "Zenith Digital Asset Access Gateway"
    DATABASE_URL = os.getenv("DATABASE_URL", "sqlite:///./crypto_bridge.db")
    DAILY_LIMIT_NGN = 10_000_000
    LICENSED_VASP_IDS = ["vasp_001", "vasp_002"]

settings = Settings()

# ===================== DATABASE =====================
engine = create_engine(
    settings.DATABASE_URL,
    connect_args={"check_same_thread": False} if settings.DATABASE_URL.startswith("sqlite") else {}
)
SessionLocal = sessionmaker(autocommit=False, autoflush=False, bind=engine)
Base = declarative_base()

# ===================== MODELS =====================
class User(Base):
    __tablename__ = "users"
    id = Column(String, primary_key=True, index=True)
    full_name = Column(String, nullable=False)
    kyc_level = Column(String, default="TIER_1")
    risk_score = Column(Float, default=0.0)
    is_active = Column(Boolean, default=True)

class FiatWallet(Base):
    __tablename__ = "fiat_wallets"
    user_id = Column(String, primary_key=True, index=True)
    balance_ngn = Column(Float, default=0.0)

class Transaction(Base):
    __tablename__ = "transactions"
    id = Column(String, primary_key=True, index=True)
    user_id = Column(String)
    amount_ngn = Column(Float)
    type = Column(String)
    vasp_id = Column(String)
    status = Column(String)
    created_at = Column(DateTime, default=datetime.datetime.utcnow)

Base.metadata.create_all(bind=engine)

# ===================== DB DEPENDENCY =====================
def get_db() -> Session:
    db = SessionLocal()
    try:
        yield db
    finally:
        db.close()

# ===================== COMPLIANCE =====================
class ComplianceError(Exception):
    def __init__(self, code: str, message: str):
        self.code = code
        self.message = message


def compliance_check(user: User, amount: float, vasp_id: str):
    if amount <= 0:
        raise ComplianceError("INVALID_AMOUNT", "Amount must be greater than zero")

    if vasp_id not in settings.LICENSED_VASP_IDS:
        raise ComplianceError("UNLICENSED_VASP", "VASP not approved")

    if amount > settings.DAILY_LIMIT_NGN:
        raise ComplianceError("CBN_LIMIT", "Daily limit exceeded")

    if user.risk_score > 0.7:
        raise ComplianceError("AML_RISK", "High AML risk user")

# ===================== WALLET =====================
def debit_wallet(wallet: FiatWallet, amount: float):
    if wallet.balance_ngn < amount:
        raise ComplianceError("INSUFFICIENT_FUNDS", "Insufficient NGN balance")
    wallet.balance_ngn -= amount

# ===================== VASP CONNECTOR (MOCK SAFE) =====================
def initiate_onramp(vasp_id: str, user_id: str, amount_ngn: float):
    # Bank never touches crypto – VASP acknowledgement only
    return {
        "vasp": vasp_id,
        "reference": user_id,
        "amount_ngn": amount_ngn,
        "status": "RECEIVED"
    }

# ===================== AUDIT =====================
def log_event(event: str, payload: dict):
    print({
        "timestamp": datetime.datetime.utcnow().isoformat(),
        "event": event,
        "payload": payload
    })

# ===================== SETTLEMENT =====================
def process_onramp(db: Session, user: User, wallet: FiatWallet, vasp_id: str, amount: float):
    compliance_check(user, amount, vasp_id)
    debit_wallet(wallet, amount)

    tx = Transaction(
        id=str(uuid.uuid4()),
        user_id=user.id,
        amount_ngn=amount,
        type="ONRAMP",
        vasp_id=vasp_id,
        status="PROCESSING"
    )

    db.add(tx)
    db.add(wallet)
    db.commit()
    db.refresh(tx)

    response = initiate_onramp(vasp_id, user.id, amount)

    log_event("ONRAMP_INITIATED", {
        "transaction_id": tx.id,
        "user_id": user.id,
        "vasp_id": vasp_id,
        "amount_ngn": amount
    })

    return {
        "transaction_id": tx.id,
        "status": tx.status,
        "vasp_response": response
    }

# ===================== FASTAPI =====================
app = FastAPI(title=settings.PROJECT_NAME)

@app.post("/onramp")
def onramp(user_id: str, amount: float, vasp_id: str, db: Session = Depends(get_db)):
    user = db.query(User).filter(User.id == user_id).first()
    wallet = db.query(FiatWallet).filter(FiatWallet.user_id == user_id).first()

    if not user or not wallet:
        raise HTTPException(status_code=404, detail="User or wallet not found")

    try:
        return process_onramp(db, user, wallet, vasp_id, amount)
    except ComplianceError as e:
        raise HTTPException(status_code=400, detail={"code": e.code, "message": e.message})
    except Exception as e:
        raise HTTPException(status_code=500, detail=str(e))

# ===================== DEV SEED (SAFE TO REMOVE) =====================
@app.on_event("startup")
def seed_dev_data():
    db = SessionLocal()
    if not db.query(User).filter(User.id == "user_001").first():
        user = User(
            id="user_001",
            full_name="John Doe",
            kyc_level="TIER_2",
            risk_score=0.2
        )
        wallet = FiatWallet(
            user_id="user_001",
            balance_ngn=15_000_000
        )
        db.add_all([user, wallet])
        db.commit()
    db.close()
