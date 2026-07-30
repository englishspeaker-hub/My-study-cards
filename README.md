# My-study-card
const express = require('express');
const multer = require('multer');
const pdfParse = require('pdf-parse');
const cors = require('cors');

const app = express();
app.use(cors()); // Allows your React app to talk to this server
app.use(express.json());

// Keep PDF in RAM for fast processing
const upload = multer({ 
  storage: multer.memoryStorage(),
  limits: { fileSize: 2 * 1024 * 1024 } // 2MB limit
});

app.post('/api/generate-cards', upload.single('document'), async (req, res) => {
  try {
    if (!req.file) return res.status(400).json({ error: "No PDF uploaded" });

    // Extract text
    const pdfData = await pdfParse(req.file.buffer);
    const rawText = pdfData.text;

    if (rawText.trim().length === 0) {
       return res.status(400).json({ error: "Could not read text. Is this a scanned image?" });
    }

    // MOCK LLM DELAY (Simulating AI processing time)
    setTimeout(() => {
      // This is the exact JSON structure your real LLM prompt should return
      const mockFlashcards = [
        { id: 1, front: "What is Spaced Repetition?", back: "A learning technique that incorporates increasing intervals of time between subsequent reviews." },
        { id: 2, front: "Why is chunking important for LLMs?", back: "It prevents exceeding token limits and improves the AI's accuracy by focusing on smaller contexts." },
        { id: 3, front: "What is an MVP?", back: "Minimum Viable Product - the leanest version of a product required to test the core concept." }
      ];
      
      res.json({ success: true, cards: mockFlashcards });
    }, 2000); 

  } catch (error) {
    console.error("Server Error:", error);
    res.status(500).json({ error: "Failed to process the PDF" });
  }
});

const PORT = 3000;
app.listen(PORT, () => console.log(`Backend running on http://localhost:${PORT}`));
import React, { useState } from 'react';

export default function App() {
  const [screen, setScreen] = useState('dashboard'); // 'dashboard', 'loading', 'editor'
  const [cards, setCards] = useState([]);
  const [deckName, setDeckName] = useState('My New Deck');

  // Handle file upload and API call
  const handleFileUpload = async (event) => {
    const file = event.target.files[0];
    if (!file) return;

    setScreen('loading');
    const formData = new FormData();
    formData.append('document', file);

    try {
      const response = await fetch('http://localhost:3000/api/generate-cards', {
        method: 'POST',
        body: formData,
      });
      const data = await response.json();
      
      if (data.cards) {
        setCards(data.cards);
        setScreen('editor');
      } else {
        alert(data.error || "Something went wrong");
        setScreen('dashboard');
      }
    } catch (error) {
      alert("Failed to connect to backend.");
      setScreen('dashboard');
    }
  };

  // Card Editor Functions
  const updateCard = (id, field, value) => {
    setCards(cards.map(c => c.id === id ? { ...c, [field]: value } : c));
  };

  const deleteCard = (id) => {
    setCards(cards.filter(c => c.id !== id));
  };

  const addBlankCard = () => {
    setCards([...cards, { id: Date.now(), front: '', back: '' }]);
  };

  // --- SCREEN RENDERERS ---

  if (screen === 'loading') {
    return (
      <div className="min-h-screen flex flex-col items-center justify-center bg-gray-50 text-gray-800">
        <div className="animate-spin text-4xl mb-4">⚙️</div>
        <h2 className="text-xl font-semibold">Reading document...</h2>
        <p className="text-gray-500 mt-2">Extracting concepts and generating cards.</p>
      </div>
    );
  }

  if (screen === 'editor') {
    return (
      <div className="max-w-4xl mx-auto p-6 text-gray-800">
        <div className="flex justify-between items-center mb-8">
          <input 
            type="text" 
            value={deckName} 
            onChange={(e) => setDeckName(e.target.value)}
            className="text-3xl font-bold bg-transparent border-b-2 border-transparent hover:border-gray-300 focus:border-blue-500 outline-none"
          />
          <div className="space-x-4">
            <button onClick={() => setScreen('dashboard')} className="text-gray-500 hover:text-gray-700">Cancel</button>
            <button onClick={() => alert("Deck Saved! (Implement DB logic here)")} className="bg-blue-600 text-white px-6 py-2 rounded-lg font-medium hover:bg-blue-700">Save Deck</button>
          </div>
        </div>

        <div className="space-y-4">
          {cards.map((card, index) => (
            <div key={card.id} className="flex gap-4 items-start bg-white p-4 rounded-xl shadow-sm border border-gray-100">
              <span className="text-gray-400 font-mono mt-2">{index + 1}</span>
              <textarea 
                value={card.front}
                onChange={(e) => updateCard(card.id, 'front', e.target.value)}
                placeholder="Front (Question)"
                className="flex-1 p-3 border rounded-lg resize-none outline-none focus:ring-2 focus:ring-blue-100"
                rows="2"
              />
              <textarea 
                value={card.back}
                onChange={(e) => updateCard(card.id, 'back', e.target.value)}
                placeholder="Back (Answer)"
                className="flex-1 p-3 border rounded-lg resize-none outline-none focus:ring-2 focus:ring-blue-100"
                rows="2"
              />
              <button onClick={() => deleteCard(card.id)} className="text-red-400 hover:text-red-600 p-2 mt-1">🗑️</button>
            </div>
          ))}
        </div>
        
        <button onClick={addBlankCard} className="mt-6 w-full py-4 border-2 border-dashed border-gray-300 text-gray-500 rounded-xl hover:bg-gray-50 hover:text-blue-600 transition-colors font-medium">
          + Add New Card
        </button>
      </div>
    );
  }

  // DEFAULT: Dashboard Screen
  return (
    <div className="min-h-screen bg-gray-50 p-8 text-gray-800">
      <header className="max-w-5xl mx-auto flex justify-between items-center mb-12">
        <h1 className="text-2xl font-bold tracking-tight text-blue-600">FlashMVP</h1>
        <div className="h-8 w-8 bg-gray-300 rounded-full"></div> {/* Mock Profile Avatar */}
      </header>

      <main className="max-w-5xl mx-auto">
        <div className="grid grid-cols-1 md:grid-cols-2 gap-6 mb-12">
          
          {/* PDF Upload Button */}
          <label className="cursor-pointer group flex flex-col items-center justify-center p-8 bg-gradient-to-br from-blue-50 to-indigo-50 border-2 border-blue-200 border-dashed rounded-2xl hover:bg-blue-100 transition-all">
            <span className="text-3xl mb-3 group-hover:scale-110 transition-transform">✨</span>
            <span className="text-lg font-semibold text-blue-800">Generate from PDF</span>
            <span className="text-sm text-blue-600/70 mt-1">Max 2MB</span>
            <input type="file" accept="application/pdf" className="hidden" onChange={handleFileUpload} />
          </label>

          {/* Empty Deck Button */}
          <button onClick={() => { setCards([]); setScreen('editor'); }} className="flex flex-col items-center justify-center p-8 bg-white border-2 border-gray-200 border-dashed rounded-2xl hover:border-gray-300 hover:bg-gray-50 transition-all">
            <span className="text-3xl mb-3 text-gray-400">+</span>
            <span className="text-lg font-semibold text-gray-600">Create Empty Deck</span>
          </button>
          
        </div>

        <h3 className="text-lg font-semibold text-gray-700 mb-4">Your Decks</h3>
        <div className="text-gray-500 italic">No decks saved yet. Create one above!</div>
      </main>
    </div>
  );
}
// Change from localhost to your Render URL:
const response = await fetch('https://my-flashcard-backend.onrender.com/api/generate-cards', {
  method: 'POST',
  body: formData,
});
