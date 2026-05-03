import { useState, useEffect } from "react";

const GOLD = "#B07D3A";
const LIGHT_GOLD = "#C8973F";

const carServices = [
  {
    category: "Interior",
    emoji: "🪑",
    items: [
      { name: "Basic Interior Clean", desc: "Vacuum, dash & vent wipe, windows, mats", price: 89, duration: 2 },
      { name: "Full Interior Detail", desc: "Deep vacuum, carpet shampoo, seat wash, full wipe-down", price: 149, duration: 3 },
      { name: "Premium Interior", desc: "Everything above + steam clean, leather condition, odor treatment", price: 199, duration: 4 },
    ],
  },
  {
    category: "Exterior",
    emoji: "✨",
    items: [
      { name: "Hand Wash & Dry", desc: "Foam pre-soak, hand wash, touchless dry, glass wipe", price: 79, duration: 1.5 },
      { name: "Exterior Detail", desc: "Hand wash, hand wax, bug & sap removal, trim dressing", price: 149, duration: 3 },
    ],
  },
  {
    category: "Wheels",
    emoji: "⚙️",
    items: [
      { name: "Wheel Wash", desc: "Rim & tire clean, brake dust removal, tire dressing", price: 39, duration: 1 },
      { name: "Wheel Detail", desc: "Degreaser foam, barrel clean, lug nuts, tire shine & protectant", price: 69, duration: 1.5 },
    ],
  },
];

const truckServices = [
  {
    category: "Interior",
    emoji: "🪑",
    items: [
      { name: "Basic Interior Clean", desc: "Vacuum cab & bed area, dash & vent wipe, windows, mats", price: 109, duration: 2.5 },
      { name: "Full Interior Detail", desc: "Deep vacuum, carpet shampoo, seat wash, full wipe-down, bed liner wipe", price: 179, duration: 3.5 },
      { name: "Premium Interior", desc: "Everything above + steam clean, leather condition, odor treatment", price: 239, duration: 4.5 },
    ],
  },
  {
    category: "Exterior",
    emoji: "✨",
    items: [
      { name: "Hand Wash & Dry", desc: "Foam pre-soak, hand wash, touchless dry, glass wipe", price: 99, duration: 2 },
      { name: "Exterior Detail", desc: "Hand wash, hand wax, bug & sap removal, trim dressing", price: 179, duration: 3.5 },
    ],
  },
  {
    category: "Wheels",
    emoji: "⚙️",
    items: [
      { name: "Wheel Wash", desc: "Rim & tire clean, brake dust removal, tire dressing (all 4–6 wheels)", price: 49, duration: 1.5 },
      { name: "Wheel Detail", desc: "Degreaser foam, barrel clean, lug nuts, tire shine & protectant", price: 89, duration: 2 },
    ],
  },
];

const TIME_SLOTS = ["8:00 AM", "9:00 AM", "10:00 AM", "11:00 AM", "12:00 PM", "1:00 PM", "2:00 PM", "3:00 PM", "4:00 PM"];

function getDaysInMonth(year, month) {
  return new Date(year, month + 1, 0).getDate();
}
function getFirstDayOfMonth(year, month) {
  return new Date(year, month, 1).getDay();
}
function isoDate(year, month, day) {
  return `${year}-${String(month + 1).padStart(2, "0")}-${String(day).padStart(2, "0")}`;
}

export default function DetailingApp() {
  const [tab, setTab] = useState("cars");
  const [step, setStep] = useState("menu"); // menu | select | calendar | form | confirm
  const [selectedService, setSelectedService] = useState(null);

  // Calendar state
  const today = new Date();
  const [calYear, setCalYear] = useState(today.getFullYear());
  const [calMonth, setCalMonth] = useState(today.getMonth());
  const [selectedDay, setSelectedDay] = useState(null);
  const [selectedSlot, setSelectedSlot] = useState(null);
  const [busySlots, setBusySlots] = useState({});
  const [loadingSlots, setLoadingSlots] = useState(false);

  // Form state
  const [form, setForm] = useState({ name: "", phone: "", email: "", notes: "" });
  const [booking, setBooking] = useState(false);
  const [booked, setBooked] = useState(false);
  const [bookingError, setBookingError] = useState("");

  const services = tab === "cars" ? carServices : truckServices;

  // Fetch busy slots for the visible month
  useEffect(() => {
    if (step !== "calendar") return;
    setLoadingSlots(true);
    const start = new Date(calYear, calMonth, 1).toISOString();
    const end = new Date(calYear, calMonth + 1, 0, 23, 59).toISOString();

    fetch("https://api.anthropic.com/v1/messages", {
      method: "POST",
      headers: { "Content-Type": "application/json" },
      body: JSON.stringify({
        model: "claude-sonnet-4-20250514",
        max_tokens: 1000,
        tools: [],
        mcp_servers: [{ type: "url", url: "https://calendarmcp.googleapis.com/mcp/v1", name: "gcal" }],
        messages: [{
          role: "user",
          content: `List all events on my primary Google Calendar from ${start} to ${end}. Return ONLY a JSON array of objects with keys "date" (YYYY-MM-DD) and "time" (like "9:00 AM"). Return nothing else, no markdown.`
        }]
      })
    })
      .then(r => r.json())
      .then(data => {
        const text = (data.content || []).filter(b => b.type === "text").map(b => b.text).join("");
        try {
          const clean = text.replace(/```json|```/g, "").trim();
          const events = JSON.parse(clean);
          const map = {};
          events.forEach(e => {
            if (!map[e.date]) map[e.date] = [];
            map[e.date].push(e.time);
          });
          setBusySlots(map);
        } catch {
          setBusySlots({});
        }
      })
      .catch(() => setBusySlots({}))
      .finally(() => setLoadingSlots(false));
  }, [step, calYear, calMonth]);

  async function handleBook() {
    if (!form.name || !form.phone) return;
    setBooking(true);
    setBookingError("");
    const dateStr = isoDate(calYear, calMonth, selectedDay);
    const hourMap = {
      "8:00 AM": "08:00", "9:00 AM": "09:00", "10:00 AM": "10:00", "11:00 AM": "11:00",
      "12:00 PM": "12:00", "1:00 PM": "13:00", "2:00 PM": "14:00", "3:00 PM": "15:00", "4:00 PM": "16:00"
    };
    const startISO = `${dateStr}T${hourMap[selectedSlot]}:00`;
    const endHour = parseInt(hourMap[selectedSlot]) + Math.ceil(selectedService.duration);
    const endISO = `${dateStr}T${String(endHour).padStart(2, "0")}:00:00`;
    const vehicleType = tab === "cars" ? "Car" : "Truck/SUV";
    const desc = `Customer: ${form.name}\nPhone: ${form.phone}\nEmail: ${form.email || "N/A"}\nVehicle: ${vehicleType}\nService: ${selectedService.name}\nPrice: $${selectedService.price}+\n${form.notes ? "Notes: " + form.notes + "\n" : ""}⚠️ Customer confirmed access to electricity & water at service location.`;

    try {
      const res = await fetch("https://api.anthropic.com/v1/messages", {
        method: "POST",
        headers: { "Content-Type": "application/json" },
        body: JSON.stringify({
          model: "claude-sonnet-4-20250514",
          max_tokens: 500,
          mcp_servers: [{ type: "url", url: "https://calendarmcp.googleapis.com/mcp/v1", name: "gcal" }],
          messages: [{
            role: "user",
            content: `Create a Google Calendar event with these exact details:
- Title: "Auto Detail: ${selectedService.name} (${vehicleType}) – ${form.name}"
- Start: ${startISO} America/Chicago
- End: ${endISO} America/Chicago
- Description: ${desc}
Confirm once created.`
          }]
        })
      });
      const data = await res.json();
      const text = (data.content || []).filter(b => b.type === "text").map(b => b.text).join("").toLowerCase();
      if (text.includes("created") || text.includes("event") || text.includes("scheduled")) {
        setBooked(true);
      } else {
        setBookingError("Booking may not have saved. Please check your Google Calendar.");
      }
    } catch {
      setBookingError("Connection error. Please try again.");
    }
    setBooking(false);
  }

  const daysInMonth = getDaysInMonth(calYear, calMonth);
  const firstDay = getFirstDayOfMonth(calYear, calMonth);
  const monthName = new Date(calYear, calMonth).toLocaleString("default", { month: "long" });
  const DAYS = ["Sun", "Mon", "Tue", "Wed", "Thu", "Fri", "Sat"];

  function isPast(day) {
    const d = new Date(calYear, calMonth, day);
    d.setHours(23, 59, 59);
    return d < new Date();
  }

  function isSlotBusy(slot) {
    const key = isoDate(calYear, calMonth, selectedDay);
    return (busySlots[key] || []).includes(slot);
  }

  // ─── BOOKED CONFIRMATION ───────────────────────────────────────────
  if (booked) {
    return (
      <div style={{ minHeight: "100vh", background: "#F7F5F2", display: "flex", alignItems: "center", justifyContent: "center", padding: "24px", fontFamily: "'Georgia', serif" }}>
        <div style={{ textAlign: "center", maxWidth: "400px" }}>
          <div style={{ fontSize: "56px", marginBottom: "16px" }}>✅</div>
          <h2 style={{ margin: "0 0 8px", fontSize: "24px", fontWeight: 400, color: "#1A1A1A" }}>You're Booked!</h2>
          <div style={{ width: "40px", height: "2px", background: GOLD, margin: "0 auto 16px" }} />
          <p style={{ color: "#666", fontFamily: "'Trebuchet MS', sans-serif", fontSize: "14px", lineHeight: 1.7, margin: "0 0 8px" }}>
            <strong>{form.name}</strong>, your appointment has been added to our calendar.
          </p>
          <div style={{ background: "#fff", border: "1px solid #E8E3DC", borderRadius: "10px", padding: "16px", margin: "20px 0", textAlign: "left" }}>
            <Row label="Service" value={selectedService.name} />
            <Row label="Vehicle" value={tab === "cars" ? "Car" : "Truck / SUV"} />
            <Row label="Date" value={`${monthName} ${selectedDay}, ${calYear}`} />
            <Row label="Time" value={selectedSlot} />
            <Row label="Price" value={`$${selectedService.price}+`} />
          </div>
          <p style={{ fontSize: "12px", color: "#AAA", fontFamily: "'Trebuchet MS', sans-serif" }}>
            We'll see you then! Questions? Call or text us at{" "}
            <a href="tel:9294761704" style={{ color: GOLD, textDecoration: "none" }}>(929) 476-1704</a>
          </p>
          <button onClick={() => { setBooked(false); setStep("menu"); setSelectedService(null); setSelectedDay(null); setSelectedSlot(null); setForm({ name: "", phone: "", email: "", notes: "" }); }}
            style={{ marginTop: "16px", background: GOLD, color: "#fff", border: "none", borderRadius: "8px", padding: "12px 28px", fontFamily: "'Trebuchet MS', sans-serif", fontSize: "13px", letterSpacing: "1px", cursor: "pointer" }}>
            BOOK ANOTHER
          </button>
        </div>
      </div>
    );
  }

  return (
    <div style={{ minHeight: "100vh", background: "#F7F5F2", fontFamily: "'Georgia', serif", paddingBottom: "60px" }}>

      {/* Header */}
      <div style={{ textAlign: "center", padding: "32px 20px 20px" }}>
        <p style={{ margin: "0 0 6px", fontSize: "11px", letterSpacing: "4px", color: GOLD, textTransform: "uppercase", fontFamily: "'Trebuchet MS', sans-serif" }}>Dallas, TX</p>
        <h1 style={{ margin: "0 0 4px", fontSize: "clamp(24px, 6vw, 38px)", fontWeight: 400, color: "#1A1A1A", letterSpacing: "1px" }}>Precision Detailing</h1>
        <p style={{ margin: "0 0 10px", fontSize: "12px", color: "#AAA", fontFamily: "'Trebuchet MS', sans-serif", letterSpacing: "2px", textTransform: "uppercase" }}>Dallas Auto Detailing</p>
        <div style={{ width: "40px", height: "2px", background: GOLD, margin: "0 auto 12px" }} />
        <a href="tel:9294761704" style={{ fontSize: "14px", color: GOLD, fontFamily: "'Trebuchet MS', sans-serif", letterSpacing: "1px", textDecoration: "none" }}>
          📞 (929) 476-1704
        </a>
      </div>

      <div style={{ maxWidth: "600px", margin: "0 auto", padding: "0 20px" }}>

        {/* ── MENU STEP ─────────────────────────────────── */}
        {step === "menu" && (
          <>
            {/* Vehicle Tabs */}
            <div style={{ display: "flex", background: "#EDEAE4", borderRadius: "8px", padding: "4px", gap: "4px", marginBottom: "24px" }}>
              {["cars", "trucks"].map(v => (
                <button key={v} onClick={() => setTab(v)} style={{
                  flex: 1, padding: "11px", border: "none", borderRadius: "6px",
                  background: tab === v ? GOLD : "transparent",
                  color: tab === v ? "#fff" : "#888",
                  fontFamily: "'Trebuchet MS', sans-serif", fontSize: "13px",
                  fontWeight: tab === v ? 700 : 400, letterSpacing: "1.5px",
                  textTransform: "uppercase", cursor: "pointer", transition: "all 0.2s"
                }}>
                  {v === "cars" ? "🚗  Cars" : "🚚  Trucks & SUVs"}
                </button>
              ))}
            </div>

            {/* Service List */}
            {services.map(section => (
              <div key={section.category} style={{ marginBottom: "24px" }}>
                <div style={{ display: "flex", alignItems: "center", gap: "10px", marginBottom: "12px" }}>
                  <span style={{ fontSize: "18px" }}>{section.emoji}</span>
                  <span style={{ fontSize: "12px", fontFamily: "'Trebuchet MS', sans-serif", fontWeight: 700, letterSpacing: "3px", textTransform: "uppercase", color: GOLD }}>{section.category}</span>
                  <div style={{ flex: 1, height: "1px", background: "#E0DBD3" }} />
                </div>
                <div style={{ display: "flex", flexDirection: "column", gap: "8px" }}>
                  {section.items.map(item => (
                    <div key={item.name} style={{ background: "#fff", border: "1px solid #E8E3DC", borderRadius: "10px", padding: "15px 18px", display: "flex", justifyContent: "space-between", alignItems: "center", gap: "12px" }}>
                      <div style={{ flex: 1 }}>
                        <div style={{ fontSize: "15px", color: "#1A1A1A", fontWeight: 600, marginBottom: "4px" }}>{item.name}</div>
                        <div style={{ fontSize: "12px", color: "#999", fontFamily: "'Trebuchet MS', sans-serif", lineHeight: 1.5 }}>{item.desc}</div>
                      </div>
                      <div style={{ textAlign: "right", flexShrink: 0 }}>
                        <div style={{ fontSize: "19px", fontWeight: 700, color: "#1A1A1A", marginBottom: "6px" }}>${item.price}+</div>
                        <button onClick={() => { setSelectedService({ ...item, vehicleType: tab }); setStep("calendar"); }}
                          style={{ background: GOLD, color: "#fff", border: "none", borderRadius: "6px", padding: "6px 14px", fontSize: "11px", fontFamily: "'Trebuchet MS', sans-serif", letterSpacing: "1px", textTransform: "uppercase", cursor: "pointer", whiteSpace: "nowrap" }}>
                          Book
                        </button>
                      </div>
                    </div>
                  ))}
                </div>
              </div>
            ))}

            <p style={{ textAlign: "center", fontSize: "11px", color: "#BBB", fontFamily: "'Trebuchet MS', sans-serif", lineHeight: 1.7 }}>
              Final price depends on vehicle condition. Tap <strong>Book</strong> to choose your date & time.
            </p>
          </>
        )}

        {/* ── CALENDAR STEP ─────────────────────────────── */}
        {step === "calendar" && (
          <>
            <button onClick={() => { setStep("menu"); setSelectedDay(null); setSelectedSlot(null); }}
              style={{ background: "none", border: "none", color: GOLD, fontFamily: "'Trebuchet MS', sans-serif", fontSize: "13px", cursor: "pointer", padding: "0 0 16px", letterSpacing: "1px" }}>
              ← Back to Menu
            </button>

            {/* Selected service reminder */}
            <div style={{ background: "#fff", border: `1px solid ${GOLD}55`, borderRadius: "10px", padding: "14px 18px", marginBottom: "20px", display: "flex", justifyContent: "space-between", alignItems: "center" }}>
              <div>
                <div style={{ fontSize: "13px", color: "#888", fontFamily: "'Trebuchet MS', sans-serif", marginBottom: "2px" }}>{tab === "cars" ? "🚗 Car" : "🚚 Truck/SUV"}</div>
                <div style={{ fontSize: "15px", fontWeight: 600, color: "#1A1A1A" }}>{selectedService?.name}</div>
              </div>
              <div style={{ fontSize: "20px", fontWeight: 700, color: GOLD }}>${selectedService?.price}+</div>
            </div>

            {/* Calendar */}
            <div style={{ background: "#fff", border: "1px solid #E8E3DC", borderRadius: "12px", padding: "20px", marginBottom: "16px" }}>
              {/* Month nav */}
              <div style={{ display: "flex", justifyContent: "space-between", alignItems: "center", marginBottom: "16px" }}>
                <button onClick={() => { if (calMonth === 0) { setCalMonth(11); setCalYear(y => y - 1); } else setCalMonth(m => m - 1); setSelectedDay(null); setSelectedSlot(null); }}
                  style={{ background: "none", border: "1px solid #E0DBD3", borderRadius: "6px", padding: "6px 12px", cursor: "pointer", color: "#888", fontSize: "14px" }}>‹</button>
                <span style={{ fontWeight: 600, fontSize: "16px", color: "#1A1A1A" }}>{monthName} {calYear}</span>
                <button onClick={() => { if (calMonth === 11) { setCalMonth(0); setCalYear(y => y + 1); } else setCalMonth(m => m + 1); setSelectedDay(null); setSelectedSlot(null); }}
                  style={{ background: "none", border: "1px solid #E0DBD3", borderRadius: "6px", padding: "6px 12px", cursor: "pointer", color: "#888", fontSize: "14px" }}>›</button>
              </div>

              {/* Day headers */}
              <div style={{ display: "grid", gridTemplateColumns: "repeat(7, 1fr)", gap: "4px", marginBottom: "6px" }}>
                {DAYS.map(d => <div key={d} style={{ textAlign: "center", fontSize: "11px", color: "#BBB", fontFamily: "'Trebuchet MS', sans-serif", fontWeight: 700, letterSpacing: "1px", padding: "4px 0" }}>{d}</div>)}
              </div>

              {/* Days grid */}
              <div style={{ display: "grid", gridTemplateColumns: "repeat(7, 1fr)", gap: "4px" }}>
                {Array(firstDay).fill(null).map((_, i) => <div key={`e${i}`} />)}
                {Array(daysInMonth).fill(null).map((_, i) => {
                  const day = i + 1;
                  const past = isPast(day);
                  const isSelected = selectedDay === day;
                  const dateKey = isoDate(calYear, calMonth, day);
                  const hasBusy = busySlots[dateKey]?.length > 0;
                  return (
                    <button key={day} disabled={past} onClick={() => { setSelectedDay(day); setSelectedSlot(null); }}
                      style={{
                        padding: "8px 4px", border: isSelected ? `2px solid ${GOLD}` : "1px solid #E8E3DC",
                        borderRadius: "8px", background: isSelected ? `${GOLD}18` : "#F9F8F6",
                        color: past ? "#CCC" : "#1A1A1A", fontSize: "13px", cursor: past ? "default" : "pointer",
                        fontFamily: "'Trebuchet MS', sans-serif", fontWeight: isSelected ? 700 : 400,
                        position: "relative", transition: "all 0.15s"
                      }}>
                      {day}
                      {hasBusy && !past && <div style={{ width: "4px", height: "4px", borderRadius: "50%", background: "#E07070", margin: "2px auto 0" }} />}
                    </button>
                  );
                })}
              </div>

              {loadingSlots && <p style={{ textAlign: "center", fontSize: "12px", color: "#BBB", fontFamily: "'Trebuchet MS', sans-serif", marginTop: "12px" }}>Checking availability…</p>}
            </div>

            {/* Time slots */}
            {selectedDay && (
              <div style={{ background: "#fff", border: "1px solid #E8E3DC", borderRadius: "12px", padding: "18px", marginBottom: "16px" }}>
                <p style={{ margin: "0 0 12px", fontSize: "13px", fontFamily: "'Trebuchet MS', sans-serif", color: "#888", letterSpacing: "1px", textTransform: "uppercase" }}>
                  Available Times — {monthName} {selectedDay}
                </p>
                <div style={{ display: "grid", gridTemplateColumns: "repeat(3, 1fr)", gap: "8px" }}>
                  {TIME_SLOTS.map(slot => {
                    const busy = isSlotBusy(slot);
                    const selected = selectedSlot === slot;
                    return (
                      <button key={slot} disabled={busy} onClick={() => setSelectedSlot(slot)}
                        style={{
                          padding: "10px 6px", border: selected ? `2px solid ${GOLD}` : "1px solid #E8E3DC",
                          borderRadius: "8px", background: busy ? "#F5F5F5" : selected ? `${GOLD}18` : "#F9F8F6",
                          color: busy ? "#CCC" : selected ? GOLD : "#1A1A1A",
                          fontSize: "13px", fontFamily: "'Trebuchet MS', sans-serif",
                          fontWeight: selected ? 700 : 400, cursor: busy ? "default" : "pointer",
                          textDecoration: busy ? "line-through" : "none", transition: "all 0.15s"
                        }}>
                        {slot}
                      </button>
                    );
                  })}
                </div>
              </div>
            )}

            {selectedSlot && (
              <button onClick={() => setStep("form")}
                style={{ width: "100%", padding: "15px", background: GOLD, color: "#fff", border: "none", borderRadius: "10px", fontSize: "14px", fontFamily: "'Trebuchet MS', sans-serif", letterSpacing: "2px", textTransform: "uppercase", fontWeight: 700, cursor: "pointer" }}>
                Continue →
              </button>
            )}
          </>
        )}

        {/* ── FORM STEP ─────────────────────────────────── */}
        {step === "form" && (
          <>
            <button onClick={() => setStep("calendar")}
              style={{ background: "none", border: "none", color: GOLD, fontFamily: "'Trebuchet MS', sans-serif", fontSize: "13px", cursor: "pointer", padding: "0 0 16px", letterSpacing: "1px" }}>
              ← Back to Calendar
            </button>

            <div style={{ background: "#fff", border: `1px solid ${GOLD}55`, borderRadius: "10px", padding: "14px 18px", marginBottom: "20px" }}>
              <div style={{ fontSize: "13px", color: "#888", fontFamily: "'Trebuchet MS', sans-serif", marginBottom: "4px" }}>
                {tab === "cars" ? "🚗 Car" : "🚚 Truck/SUV"} · {selectedService?.name}
              </div>
              <div style={{ fontSize: "14px", color: "#555", fontFamily: "'Trebuchet MS', sans-serif" }}>
                📅 {monthName} {selectedDay}, {calYear} at {selectedSlot}
              </div>
            </div>

            {/* Requirements notice */}
            <div style={{ background: "#FFFBF2", border: "1px solid #F0DFA0", borderRadius: "10px", padding: "14px 16px", marginBottom: "4px", display: "flex", gap: "10px", alignItems: "flex-start" }}>
              <span style={{ fontSize: "18px", flexShrink: 0 }}>⚠️</span>
              <div>
                <div style={{ fontSize: "12px", fontFamily: "'Trebuchet MS', sans-serif", fontWeight: 700, color: "#8A6A00", letterSpacing: "0.5px", marginBottom: "4px" }}>Required at your location</div>
                <div style={{ fontSize: "12px", fontFamily: "'Trebuchet MS', sans-serif", color: "#AA8800", lineHeight: 1.6 }}>
                  Please ensure access to <strong>electricity</strong> and <strong>water</strong> at the service location before your appointment.
                </div>
              </div>
            </div>

            <div style={{ display: "flex", flexDirection: "column", gap: "12px" }}>
              {[
                { key: "name", label: "Your Name *", placeholder: "John Smith", type: "text" },
                { key: "phone", label: "Phone Number *", placeholder: "(214) 555-0123", type: "tel" },
                { key: "email", label: "Email (optional)", placeholder: "john@email.com", type: "email" },
                { key: "notes", label: "Notes (optional)", placeholder: "e.g. pet hair, heavy stains, specific areas…", type: "text" },
              ].map(f => (
                <div key={f.key}>
                  <label style={{ display: "block", fontSize: "11px", fontFamily: "'Trebuchet MS', sans-serif", color: "#AAA", letterSpacing: "1px", textTransform: "uppercase", marginBottom: "6px" }}>{f.label}</label>
                  <input
                    type={f.type}
                    placeholder={f.placeholder}
                    value={form[f.key]}
                    onChange={e => setForm(p => ({ ...p, [f.key]: e.target.value }))}
                    style={{ width: "100%", padding: "12px 14px", border: "1px solid #E8E3DC", borderRadius: "8px", fontSize: "14px", fontFamily: "'Trebuchet MS', sans-serif", color: "#1A1A1A", background: "#fff", outline: "none", boxSizing: "border-box" }}
                  />
                </div>
              ))}
            </div>

            {bookingError && <p style={{ color: "#E07070", fontSize: "13px", fontFamily: "'Trebuchet MS', sans-serif", marginTop: "12px" }}>{bookingError}</p>}

            <button
              onClick={handleBook}
              disabled={booking || !form.name || !form.phone}
              style={{
                width: "100%", marginTop: "20px", padding: "15px",
                background: booking || !form.name || !form.phone ? "#CCC" : GOLD,
                color: "#fff", border: "none", borderRadius: "10px",
                fontSize: "14px", fontFamily: "'Trebuchet MS', sans-serif",
                letterSpacing: "2px", textTransform: "uppercase", fontWeight: 700,
                cursor: booking || !form.name || !form.phone ? "default" : "pointer"
              }}>
              {booking ? "Booking…" : "Confirm Booking"}
            </button>
          </>
        )}

      </div>
    </div>
  );
}

function Row({ label, value }) {
  return (
    <div style={{ display: "flex", justifyContent: "space-between", padding: "6px 0", borderBottom: "1px solid #F0EDE8", fontSize: "13px", fontFamily: "'Trebuchet MS', sans-serif" }}>
      <span style={{ color: "#AAA" }}>{label}</span>
      <span style={{ color: "#1A1A1A", fontWeight: 600 }}>{value}</span>
    </div>
  );
}
