// App de Finanças Pessoais - Versão Web com React + Firebase + Gráficos
// Funcionalidades: Cadastro/login, entradas/saídas, saldo, gráficos

import React, { useEffect, useState } from 'react';
import { initializeApp } from 'firebase/app';
import {
  getAuth,
  onAuthStateChanged,
  signInWithEmailAndPassword,
  createUserWithEmailAndPassword,
  signOut,
} from 'firebase/auth';
import {
  getFirestore,
  collection,
  addDoc,
  getDocs,
  query,
  where,
} from 'firebase/firestore';
import {
  VictoryPie,
  VictoryChart,
  VictoryLine,
  VictoryTheme,
} from 'victory';
import './App.css';

const firebaseConfig = {
  apiKey: "YOUR_API_KEY",
  authDomain: "YOUR_AUTH_DOMAIN",
  projectId: "YOUR_PROJECT_ID",
  storageBucket: "YOUR_STORAGE_BUCKET",
  messagingSenderId: "YOUR_SENDER_ID",
  appId: "YOUR_APP_ID",
};

const app = initializeApp(firebaseConfig);
const auth = getAuth(app);
const db = getFirestore(app);

function App() {
  const [user, setUser] = useState(null);
  const [email, setEmail] = useState('');
  const [password, setPassword] = useState('');
  const [amount, setAmount] = useState('');
  const [type, setType] = useState('entrada');
  const [transactions, setTransactions] = useState([]);

  useEffect(() => {
    const unsubscribe = onAuthStateChanged(auth, (user) => {
      setUser(user);
      if (user) loadTransactions(user.uid);
    });
    return unsubscribe;
  }, []);

  const register = async () => {
    try {
      await createUserWithEmailAndPassword(auth, email, password);
    } catch (e) {
      alert(e.message);
    }
  };

  const login = async () => {
    try {
      await signInWithEmailAndPassword(auth, email, password);
    } catch (e) {
      alert(e.message);
    }
  };

  const logout = async () => {
    await signOut(auth);
  };

  const addTransaction = async () => {
    if (!amount) return;
    const doc = {
      userId: user.uid,
      amount: parseFloat(amount),
      type,
      date: new Date(),
    };
    await addDoc(collection(db, 'transactions'), doc);
    setAmount('');
    loadTransactions(user.uid);
  };

  const loadTransactions = async (uid) => {
    const q = query(collection(db, 'transactions'), where('userId', '==', uid));
    const snapshot = await getDocs(q);
    const data = snapshot.docs.map((doc) => doc.data());
    setTransactions(data);
  };

  const saldo = transactions.reduce(
    (acc, t) => (t.type === 'entrada' ? acc + t.amount : acc - t.amount),
    0
  );

  const dataPie = [
    {
      x: 'Entradas',
      y: transactions
        .filter((t) => t.type === 'entrada')
        .reduce((s, t) => s + t.amount, 0),
    },
    {
      x: 'Saídas',
      y: transactions
        .filter((t) => t.type === 'saída')
        .reduce((s, t) => s + t.amount, 0),
    },
  ];

  return (
    <div className="App">
      {!user ? (
        <div className="login-box">
          <h2>Login / Registro</h2>
          <input
            placeholder="Email"
            value={email}
            onChange={(e) => setEmail(e.target.value)}
          />
          <input
            placeholder="Senha"
            type="password"
            value={password}
            onChange={(e) => setPassword(e.target.value)}
          />
          <button onClick={login}>Entrar</button>
          <button onClick={register}>Registrar</button>
        </div>
      ) : (
        <div className="dashboard">
          <h2>Bem-vindo! Saldo: R$ {saldo.toFixed(2)}</h2>
          <input
            placeholder="Valor"
            type="number"
            value={amount}
            onChange={(e) => setAmount(e.target.value)}
          />
          <div className="btn-group">
            <button onClick={() => { setType('entrada'); addTransaction(); }}>Entrada</button>
            <button onClick={() => { setType('saída'); addTransaction(); }}>Saída</button>
          </div>
          <button onClick={logout} className="logout">Sair</button>

          <h3>Gráfico de Gastos</h3>
          <VictoryPie data={dataPie} colorScale={["green", "red"]} />

          <h3>Histórico</h3>
          <ul>
            {transactions.map((t, i) => (
              <li key={i}>
                {t.type.toUpperCase()}: R$ {t.amount.toFixed(2)}
              </li>
            ))}
          </ul>
        </div>
      )}
    </div>
  );
}

export default App;
