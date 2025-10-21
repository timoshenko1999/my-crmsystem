import React, { useEffect, useState } from 'react';

export default function KeyPickupRegistration() {
  const MAX_PER_DAY = 20;
  const ADMIN_PASS = 'Lugovaya7'; // установлен пользователем

  const [name, setName] = useState('');
  const [phone, setPhone] = useState('');
  const [email, setEmail] = useState('');
  const [date, setDate] = useState('');
  const [note, setNote] = useState('');
  const [message, setMessage] = useState('');

  const [registrations, setRegistrations] = useState({});
  const [view, setView] = useState('form');
  const [adminPassInput, setAdminPassInput] = useState('');

  useEffect(() => {
    const params = new URLSearchParams(window.location.search);
    const preName = params.get('name') || '';
    const preEmail = params.get('email') || '';
    const preDate = params.get('date') || '';
    if (preName) setName(preName);
    if (preEmail) setEmail(preEmail);
    if (preDate) setDate(preDate);
  }, []);

  useEffect(() => {
    const raw = localStorage.getItem('key_pickup_regs');
    if (raw) {
      try {
        setRegistrations(JSON.parse(raw));
      } catch (e) {
        setRegistrations({});
      }
    }
  }, []);

  function countForDate(d) {
    if (!d) return 0;
    return (registrations[d] || []).length;
  }

  function saveRegistrations(newRegs) {
    setRegistrations(newRegs);
    localStorage.setItem('key_pickup_regs', JSON.stringify(newRegs));
  }

  function handleSubmit(e) {
    e.preventDefault();
    setMessage('');
    if (!name.trim() || !phone.trim() || !date) {
      setMessage('Пожалуйста, заполните имя, телефон и выберите дату.');
      return;
    }
    const todayStr = new Date().toISOString().slice(0, 10);
    if (date < todayStr) {
      setMessage('Нельзя выбрать прошедшую дату.');
      return;
    }
    const currentCount = countForDate(date);
    if (currentCount >= MAX_PER_DAY) {
      setMessage('Извините, на выбранную дату все слоты заняты.');
      return;
    }
    const entry = {
      id: Date.now(),
      name: name.trim(),
      phone: phone.trim(),
      email: email.trim(),
      note: note.trim(),
      date,
      createdAt: new Date().toISOString(),
    };
    const newRegs = { ...registrations };
    if (!newRegs[date]) newRegs[date] = [];
    newRegs[date].push(entry);
    saveRegistrations(newRegs);
    setMessage('Спасибо — вы успешно записаны! Мы отправим подтверждение.');
    setName('');
    setPhone('');
    setEmail('');
    setNote('');
  }

  function generateShareLink() {
    const base = window.location.origin + window.location.pathname;
    const params = new URLSearchParams();
    if (name) params.set('name', name);
    if (email) params.set('email', email);
    if (date) params.set('date', date);
    return base + (params.toString() ? ('?' + params.toString()) : '');
  }

  function handleAdminLogin(e) {
    e.preventDefault();
    if (adminPassInput === ADMIN_PASS) {
      setView('admin');
      setAdminPassInput('');
    } else {
      setMessage('Неверный пароль администратора.');
    }
  }

  function exportCSV() {
    const rows = [];
    Object.keys(registrations).forEach(d => {
      registrations[d].forEach(r => {
        rows.push({ date: d, name: r.name, phone: r.phone, email: r.email, note: r.note, createdAt: r.createdAt });
      });
    });
    const header = ['date', 'name', 'phone', 'email', 'note', 'createdAt'];
    const csv = [header.join(',')].concat(rows.map(r => header.map(h => '"' + (r[h] || '').replace(/"/g, '""') + '"').join(','))).join('\n');
    const blob = new Blob([csv], { type: 'text/csv;charset=utf-8;' });
    const url = URL.createObjectURL(blob);
    const a = document.createElement('a');
    a.href = url;
    a.download = 'registrations.csv';
    a.click();
    URL.revokeObjectURL(url);
  }

  function clearAll() {
    if (!confirm('Вы уверены, что хотите удалить все записи? Это действие нельзя отменить.')) return;
    saveRegistrations({});
  }

  const minDate = new Date().toISOString().slice(0, 10);

  return (
    <div className="min-h-screen bg-black text-white flex items-center justify-center p-4">
      <div className="max-w-3xl w-full bg-neutral-900 shadow-lg rounded-2xl p-6 border border-gray-700">
        <header className="flex items-center justify-between mb-6 border-b border-gray-700 pb-3">
          <div>
            <h1 className="text-2xl font-bold text-white">ЖК «Портрет» — выдача ключей</h1>
            <p className="text-sm text-gray-400 mt-1">Максимум {MAX_PER_DAY} человек в день. Регистрируйтесь заранее.</p>
          </div>
          <div className="text-right">
            <button onClick={() => setView('form')} className="text-sm mr-2 underline">Форма</button>
            <button onClick={() => setView('login')} className="text-sm underline">Админ</button>
          </div>
        </header>

        {view === 'form' && (
          <main>
            <form onSubmit={handleSubmit} className="grid grid-cols-1 gap-4">
              <div className="grid grid-cols-2 gap-4">
                <label className="flex flex-col">
                  <span className="text-sm">Имя*</span>
                  <input value={name} onChange={e => setName(e.target.value)} className="bg-neutral-800 border border-gray-600 rounded px-3 py-2 text-white" placeholder="Иван Иванов" />
                </label>
                <label className="flex flex-col">
                  <span className="text-sm">Телефон*</span>
                  <input value={phone} onChange={e => setPhone(e.target.value)} className="bg-neutral-800 border border-gray-600 rounded px-3 py-2 text-white" placeholder="+7 9xx xxx xxxx" />
                </label>
              </div>

              <div className="grid grid-cols-2 gap-4">
                <label className="flex flex-col">
                  <span className="text-sm">Email</span>
                  <input value={email} onChange={e => setEmail(e.target.value)} className="bg-neutral-800 border border-gray-600 rounded px-3 py-2 text-white" placeholder="name@example.com" />
                </label>
                <label className="flex flex-col">
                  <span className="text-sm">Дата получения*</span>
                  <input type="date" value={date} min={minDate} onChange={e => setDate(e.target.value)} className="bg-neutral-800 border border-gray-600 rounded px-3 py-2 text-white" />
                </label>
              </div>

              <label className="flex flex-col">
                <span className="text-sm">Примечание</span>
                <textarea value={note} onChange={e => setNote(e.target.value)} className="bg-neutral-800 border border-gray-600 rounded px-3 py-2 text-white" placeholder="например, номер договора, подъезд..." />
              </label>

              <div className="flex items-center justify-between mt-2">
                <div className="text-sm text-gray-400">Свободно на выбранную дату: <strong>{date ? (MAX_PER_DAY - countForDate(date)) : '-'}</strong></div>
                <div className="flex gap-2">
                  <button type="submit" className="bg-white text-black px-4 py-2 rounded font-semibold">Записаться</button>
                  <button type="button" onClick={() => { navigator.clipboard?.writeText(generateShareLink()); setMessage('Ссылка скопирована'); }} className="border border-gray-500 px-4 py-2 rounded">Копировать ссылку</button>
                </div>
              </div>

              {message && <div className="mt-2 text-sm text-gray-300">{message}</div>}
            </form>
          </main>
        )}

        {view === 'login' && (
          <div>
            <h2 className="font-semibold">Вход администратора</h2>
            <form onSubmit={handleAdminLogin} className="mt-4 grid grid-cols-1 gap-2 max-w-sm">
              <input value={adminPassInput} onChange={e => setAdminPassInput(e.target.value)} type="password" placeholder="Пароль" className="bg-neutral-800 border border-gray-600 rounded px-3 py-2 text-white" />
              <div className="flex gap-2">
                <button className="bg-white text-black px-4 py-2 rounded font-semibold">Войти</button>
                <button type="button" onClick={() => setView('form')} className="border border-gray-500 px-4 py-2 rounded">Отмена</button>
              </div>
            </form>
            {message && <div className="mt-2 text-sm text-red-400">{message}</div>}
          </div>
        )}

        {view === 'admin' && (
          <div className="mt-4">
            <div className="flex items-center justify-between">
              <h2 className="font-semibold text-white">Панель администратора</h2>
              <div className="flex gap-2">
                <button onClick={exportCSV} className="border border-gray-500 px-3 py-2 rounded">Экспорт CSV</button>
                <button onClick={() => { setView('form'); }} className="border border-gray-500 px-3 py-2 rounded">Вернуться</button>
                <button onClick={clearAll} className="border border-red-600 px-3 py-2 rounded text-red-400">Очистить</button>
              </div>
            </div>

            <div className="mt-4 grid gap-4">
              {Object.keys(registrations).length === 0 && <div className="text-sm text-gray-500">Пока нет записей.</div>}
              {Object.keys(registrations).sort().map(d => (
                <div key={d} className="border border-gray-700 rounded p-3 bg-neutral-800">
                  <div className="flex items-center justify-between">
                    <div>
                      <div className="font-medium text-white">{d}</div>
                      <div className="text-sm text-gray-400">{registrations[d].length} / {MAX_PER_DAY}</div>
                    </div>
                  </div>

                  <ul className="mt-3 space-y-2">
                    {registrations[d].map(r => (
                      <li key={r.id} className="text-sm text-gray-300">
                        <div><strong>{r.name}</strong> — {r.phone} {r.email ? `— ${r.email}` : ''}</div>
                        {r.note && <div className="text-gray-400">{r.note}</div>}
                        <div className="text-xs text-gray-500">{new Date(r.createdAt).toLocaleString()}</div>
                      </li>
                    ))}
                  </ul>
                </div>
              ))}
            </div>
          </div>
        )}

        <footer className="text-xs text-gray-500 mt-6 border-t border-gray-700 pt-2">ЖК «Портрет» — автоматическая запись на выдачу ключей. Данные хранятся локально.</footer>
      </div>
    </div>
  );
}
