whatsapp-clone/
├── backend/
│   └── server.js
└── frontend/
    └── App.js
    # Whatsap-clone
import React, { useState, useEffect, useRef } from 'react';
import io from 'socket.io-client';

const socket = io('http://localhost:4000');

export default function App() {
  const [username, setUsername] = useState('');
  const [loggedIn, setLoggedIn] = useState(false);
  const [input, setInput] = useState('');
  const [messages, setMessages] = useState([]);
  const messageRefs = useRef({});

  useEffect(() => {
    socket.on('previous_messages', (msgs) => setMessages(msgs));

    socket.on('receive_message', (message) => {
      setMessages(prev => [...prev, message]);
    });

    socket.on('message_read_update', ({ messageId, username: reader }) => {
      setMessages(prev =>
        prev.map(msg => {
          if (msg.id === messageId && !msg.readBy.includes(reader)) {
            return { ...msg, readBy: [...msg.readBy, reader] };
          }
          return msg;
        })
      );
    });

    return () => {
      socket.off('previous_messages');
      socket.off('receive_message');
      socket.off('message_read_update');
    };
  }, []);

  useEffect(() => {
    const observer = new IntersectionObserver(entries => {
      entries.forEach(entry => {
        if (entry.isIntersecting) {
          const id = entry.target.getAttribute('data-id');
          socket.emit('message_read', { messageId: id, username });
        }
      });
    }, { threshold: 0.7 });

    Object.values(messageRefs.current).forEach(el => {
      if (el) observer.observe(el);
    });

    return () => observer.disconnect();
  }, [messages, username]);

  const sendMessage = () => {
    if (input.trim() === '') return;
    socket.emit('send_message', { sender: username, content: input });
    setInput('');
  };

  if (!loggedIn) {
    return (
      <div style={{ padding: 20 }}>
        <h2>Giriş Yap</h2>
        <input
          placeholder="Kullanıcı Adı"
          value={username}
          onChange={e => setUsername(e.target.value)}
        />
        <button onClick={() => username.trim() && setLoggedIn(true)}>Giriş</button>
      </div>
    );
  }

  return (
    <div style={{ maxWidth: 600, margin: '20px auto', fontFamily: 'Arial, sans-serif' }}>
      <h3>Hoşgeldin, {username}</h3>

      <div style={{
        border: '1px solid #ccc', padding: 10, height: 400,
        overflowY: 'auto', marginBottom: 10,
        display: 'flex', flexDirection: 'column',
      }}>
        {messages.map(msg => (
          <div
            key={msg.id}
            data-id={msg.id}
            ref={el => (messageRefs.current[msg.id] = el)}
            style={{
              alignSelf: msg.sender === username ? 'flex-end' : 'flex-start',
              backgroundColor: msg.sender === username ? '#dcf8c6' : '#fff',
              marginBottom: 8,
              padding: 10,
              borderRadius: 10,
              maxWidth: '80%',
              position: 'relative',
            }}
          >
            <strong>{msg.sender}</strong>
            <p style={{ margin: '5px 0' }}>{msg.content}</p>
            <small style={{ fontSize: 10, color: '#999' }}>
              {new Date(msg.timestamp).toLocaleTimeString()}
              {msg.sender === username && (
                <span style={{ marginLeft: 8, color: 'green', fontWeight: 'bold' }}>
                  {msg.readBy.length > 1 ? '✓✓' : '✓'}
                </span>
              )}
            </small>
          </div>
        ))}
      </div>

      <input
        type="text"
        placeholder="Mesaj yaz..."
        value={input}
        onChange={e => setInput(e.target.value)}
        onKeyDown={e => e.key === 'Enter' && sendMessage()}
        style={{ width: '80%', padding: 10 }}
      />
      <button onClick={sendMessage} style={{ padding: '10px 15px', marginLeft: 10 }}>
        Gönder
      </button>
    </div>
  );
}
