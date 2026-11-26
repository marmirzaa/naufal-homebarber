// Slot setiap 30 min, 10:00 - 22:00 (ubah OPEN / CLOSE kalau nak)
const SV = { kids: 15, normal: 20, longtrim: 25 }, OPEN = 10, CLOSE = 22;
const J = (o, s = 200) => new Response(JSON.stringify(o), { status: s, headers: { 'content-type': 'application/json' } });
const TIMES = (() => { const a = []; for (let m = OPEN * 60; m < CLOSE * 60; m += 30) a.push(String(Math.floor(m / 60)).padStart(2, '0') + ':' + (m % 60 ? '30' : '00')); return a; })();
const now = () => new Date(Date.now() + 288e5).toISOString().slice(0, 16); // waktu Malaysia
const dOk = d => /^\d{4}-\d\d-\d\d$/.test(d || '');
const digits = s => { const d = String(s || '').replace(/\D/g, ''); return d.startsWith('0') ? '6' + d : d; };

async function day(db, date) {
  const [b, k] = await Promise.all([
    db.prepare('SELECT time FROM blocked WHERE date=?').bind(date).all(),
    db.prepare("SELECT id,time,service,phone,name FROM bookings WHERE date=? AND status='active'").bind(date).all(),
  ]);
  const off = new Set(b.results.map(r => r.time)), taken = new Map(k.results.map(r => [r.time, r])), n = now();
  return {
    closed: off.has('*'),
    slots: TIMES.map(t => ({ t, s: date + 'T' + t <= n ? 'past' : off.has('*') || off.has(t) ? 'off' : taken.has(t) ? 'taken' : 'free', b: taken.get(t) || null })),
  };
}

export async function onRequest({ request, env }) {
  const url = new URL(request.url), p = url.pathname.replace(/^\/api\/?/, ''), m = request.method, db = env.DB, q = url.searchParams;
  const b = m === 'POST' ? await request.json().catch(() => ({})) : {};
  try {
    if (p.startsWith('admin/')) {
      if (!env.ADMIN_PASSWORD || request.headers.get('x-admin') !== env.ADMIN_PASSWORD) return J({ error: 'Kata laluan salah' }, 401);
      if (p === 'admin/day') { if (!dOk(q.get('date'))) return J({ error: 'bad date' }, 400); return J(await day(db, q.get('date'))); }
      if (p === 'admin/upcoming') {
        const r = await db.prepare("SELECT * FROM bookings WHERE status='active' AND date||'T'||time>? ORDER BY date,time").bind(now()).all();
        return J({ bookings: r.results });
      }
      if (p === 'admin/toggle') {
        if (!dOk(b.date) || !(b.time === '*' || TIMES.includes(b.time))) return J({ error: 'bad' }, 400);
        const d = await db.prepare('DELETE FROM blocked WHERE date=? AND time=?').bind(b.date, b.time).run();
        if (!d.meta.changes) await db.prepare('INSERT INTO blocked VALUES(?,?)').bind(b.date, b.time).run();
        return J({ ok: 1 });
      }
      if (p === 'admin/cancel') { await db.prepare("UPDATE bookings SET status='cancelled' WHERE id=?").bind(b.id).run(); return J({ ok: 1 }); }
      if (p === 'admin/gallery') {
        if (!(b.slot >= 1 && b.slot <= 6) || !String(b.img).startsWith('data:image/') || b.img.length > 600000) return J({ error: 'Gambar tak sah atau terlalu besar' }, 400);
        await db.prepare('INSERT OR REPLACE INTO gallery VALUES(?,?)').bind(b.slot, b.img).run();
        return J({ ok: 1 });
      }
    }
    if (p === 'slots') {
      if (!dOk(q.get('date'))) return J({ error: 'bad date' }, 400);
      const d = await day(db, q.get('date'));
      return J({ closed: d.closed, slots: d.slots.map(({ t, s }) => ({ t, s })) });
    }
    if (p === 'gallery') { const r = await db.prepare('SELECT slot,img FROM gallery').all(); return J({ images: r.results }); }
    if (p === 'book' && m === 'POST') {
      const ph = digits(b.phone);
      if (!Object.hasOwn(SV, b.service) || !dOk(b.date) || !TIMES.includes(b.time) || ph.length < 10 || ph.length > 13) return J({ error: 'Maklumat tak lengkap. Semak nombor telefon.' }, 400);
      const s = (await day(db, b.date)).slots.find(x => x.t === b.time);
      if (s.s !== 'free') return J({ error: 'Slot ni dah tak available. Pilih masa lain.' }, 409);
      try {
        const r = await db.prepare('INSERT INTO bookings(service,date,time,phone,name) VALUES(?,?,?,?,?)').bind(b.service, b.date, b.time, ph, String(b.name || '').slice(0, 40)).run();
        return J({ ok: 1, id: r.meta.last_row_id });
      } catch (e) { return J({ error: 'Slot ni baru je diambil orang lain.' }, 409); }
    }
    if (p === 'mine') {
      const ph = digits(q.get('phone')); if (ph.length < 10) return J({ bookings: [] });
      const r = await db.prepare("SELECT id,service,date,time FROM bookings WHERE phone=? AND status='active' AND date||'T'||time>? ORDER BY date,time").bind(ph, now()).all();
      return J({ bookings: r.results });
    }
    if (p === 'cancel' && m === 'POST') {
      const r = await db.prepare("UPDATE bookings SET status='cancelled' WHERE id=? AND phone=? AND status='active'").bind(b.id, digits(b.phone)).run();
      return J({ ok: r.meta.changes > 0 });
    }
    return J({ error: 'Not found' }, 404);
  } catch (e) { return J({ error: 'Server error' }, 500); }
}
