import React, { useEffect, useMemo, useState } from "react";

export default function SaringaApp() {
  // === Configuración de negocio ===
  const ML_POR_PERSONA = 400; // 400 ml por persona
  const LITROS_POR_20 = 5; // 5 litros => 20 personas
  const PERSONAS_POR_5_L = 20;

  const MIN_LITROS = 10;
  const MAX_LITROS = 150;

  // Precio por litro:
  // 20-25 L => 150
  // > 25 L => 130
  // Mínimo: 130 (ya cubierto con >25 L)
  function precioPorLitro(litros) {
    if (litros >= 20 && litros <= 25) return 150;
    if (litros > 25) return 130;
    // Para 10-19 L: no definiste explícitamente,
    // para no romper tu regla de mínimo/máximo,
    // usamos 150 como tarifa base.
    return 150;
  }

  // Para el control por deslizador:
  // "cada 5 litros alcanza para 20 personas"
  // Entonces, personas -> litros = personas / 20 * 5 = personas * 0.25
  // (1 persona = 0.25 L)
  const litersFromPeople = (people) => people * (LITROS_POR_20 / PERSONAS_POR_20); // people * 0.25

  // Redondeo:
  // - Los litros NO pueden cambiar de tamaño según el negocio.
  // Tomamos litros en múltiplos de 0.25 L (que corresponde a 1 persona).
  // Aun así, aplicamos límite y redondeo para UI.
  function clamp(n, min, max) {
    return Math.max(min, Math.min(max, n));
  }

  // === UI: Estado ===
  const [mode, setMode] = useState("personas"); // "personas" | "litros"
  const [people, setPeople] = useState(20); // valor inicial
  const [liters, setLiters] = useState(() => Math.round(litersFromPeople(20) * 100) / 100);

  // Campo de sabores:
  const sabores = useMemo(
    () => [
      "Queso",
      "Guayaba",
      "Mango-fresa",
      "Mango",
      "Limón",
      "Mango-chile",
      "Choco-platano",
      "Plátano con chispas de chocolate",
      "Crema de limón",
      "Beso de angel",
      "Cajeta",
      "Rompope",
      "Coco",
      "Vainilla",
      "Fresas con crema",
      "Nogada",
      "Naranja-limon",
      "Bailey's",
      "Mazapán",
      "Aguacate",
      "Mamey",
      "Guanabana",
      "Tamarindo",
      "Ron con pasas",
      "Melón",
      "Cereza",
      "Mantecado",
      "Pistache",
      "Pepino-Limon",
      "Manzana-pepino-limon",
      "Tequila-limon",
      "Mojito",
    ],
    []
  );

  // Selección de sabores:
  // "cada sabor suma 5 litros" => saboresSeleccionados * 5 = litrosAsignados
  const [selectedFlavors, setSelectedFlavors] = useState([]);
  const flavorsLiters = selectedFlavors.length * 5; // litros que "cuestan" los sabores
  const [applyFlavorRule, setApplyFlavorRule] = useState(true);

  // === Sincronización entre personas y litros ===
  useEffect(() => {
    if (mode === "personas") {
      const l = litersFromPeople(people);
      setLiters(l);
    }
  }, [mode, people]);

  useEffect(() => {
    if (mode === "litros") {
      // Al cambiar litros, ajustamos personas aproximadas (inversa)
      const p = liters / (LITROS_POR_20 / PERSONAS_POR_20); // litros / 0.25 = personas
      setPeople(Math.round(p));
    }
  }, [mode, liters]);

  // === Aplicación de límites ===
  useEffect(() => {
    // Limitamos litros calculados por personas para no exceder.
    // Pero si activaste la regla de sabores, los sabores dominan el cálculo.
    if (applyFlavorRule && selectedFlavors.length > 0) {
      const l = flavorsLiters;
      const clamped = clamp(l, MIN_LITROS, MAX_LITROS);
      // Si clamped cambia, también actualizamos personas.
      setLiters(clamped);
      const p = Math.round(clamped / 0.25); // 0.25 L por persona
      setPeople(p);
      return;
    }

    // Sin sabores dominando:
    const l = litersFromPeople(people);
    const clamped = clamp(Math.round(l * 4) / 4, MIN_LITROS, MAX_LITROS); // step 0.25L
    setLiters(clamped);
    const p = Math.round(clamped / 0.25);
    setPeople(p);
  }, [applyFlavorRule, selectedFlavors.length, flavorsLiters, people]);

  const precioLitro = useMemo(() => precioPorLitro(liters), [liters]);
  const subtotal = useMemo(() => litrosSafe(liters) * precioLitro, [liters, precioLitro]);

  function litrosSafe(x) {
    if (!Number.isFinite(x)) return 0;
    return clamp(x, MIN_LITROS, MAX_LITROS);
  }

  // === Entregas y pagos ===
  const phone = "7711918713";
  const whatsappLink = `https://wa.me/52${phone}?text=${encodeURIComponent(
    `Hola Saringa 🍧🦋 Quiero cotizar nieve.\n` +
      `Modo: ${mode}\n` +
      `Personas: ${people}\n` +
      `Litros estimados: ${Math.round(liters * 100) / 100}\n` +
      `Sabores (${selectedFlavors.length}): ${selectedFlavors.join(", ") || "Ninguno"}\n` +
      `Me interesa acordar ubicación en Hidalgo (1 semana de anticipación, 2 horas de servicio).\n` +
      `Gracias!`
  )}`;

  // === Selección de sabores ===
  function toggleFlavor(name) {
    setSelectedFlavors((prev) => {
      const has = prev.includes(name);
      const next = has ? prev.filter((x) => x !== name) : [...prev, name];
      return next;
    });
  }

  // === Tablas rápidas de reglas ===
  const infoReglas = useMemo(() => {
    const ml = ML_POR_PERSONA;
    const litrosPerPerson = ml / 1000; // 0.4? (pero tu regla manda 400 ml => 1 persona = 0.4 L)
    // Sin embargo, con tu regla "5 L para 20 personas", eso implica 0.25 L por persona.
    // Priorizamos tu relación explícita: 5 L = 20 personas.
    // En el UI dejamos 400 ml por persona como texto comercial y usamos 5L/20 personas para cálculos.
    return {
      texto: `Anunciamos: 400 ml por persona. Cálculo comercial basado en: 5 litros para 20 personas (0.25 L/persona).`,
      litrosPerPerson: litrosPerPerson,
    };
  }, []);

  // Validaciones:
  const litrosRedondeados = Math.round(litros * 100) / 100;
  const litrosMaxByRule = MAX_LITROS;
  const saboresDentroDeLimite = flavorsLiters <= litrosMaxByRule;

  return (
    <div style={styles.page}>
      {/* Header */}
      <header style={styles.header}>
        <div style={styles.brandWrap}>
          <img
            src="/logo.png"
            alt="Logo Saringa"
            style={styles.logo}
          />
          <div style={styles.brandText}>
            <div style={styles.brandName}>Saringa</div>
            <div style={styles.brandSub}>
              Nieves 100% naturales, artesanales 🥄🍧
            </div>
          </div>
        </div>

        <div style={styles.contactBox}>
          <div style={styles.contactTitle}>📍 Hidalgo, México</div>
          <a className="phoneLink" href={`tel:${phone}`} style={styles.phoneLink}>
            📞 {phone}
          </a>
          <a className="whatsLink" href={whatsappLink} target="_blank" rel="noreferrer" style={styles.whatsBtn}>
            💬 Cotizar por WhatsApp
          </a>
          <div style={styles.contactHint}>
            *Para pagos y acuerdos, siempre contáctanos al número de arriba.
          </div>
        </div>
      </header>

      {/* Hero */}
      <section style={styles.hero}>
        <div style={styles.heroLeft}>
          <h2 style={styles.heroTitle}>Nieves personalizadas para tus eventos 🎉</h2>
          <p style={styles.heroText}>
            Encargos con <b>1 semana</b> de anticipación. Servicio con <b>2 horas</b>, vasos y cucharas para pruebas
            (material incluido). Entrega en <b>Hidalgo</b> únicamente.
          </p>
          <p style={styles.heroText}>
            Límite por encargo: <b>mínimo 10 L</b> y <b>máximo 150 L</b>.
          </p>

          <div style={styles.notice}>
            <span>⚠️</span>
            <div>
              Los litros y sabores siguen reglas fijas: <b>cada sabor agrega 5 litros</b> (no modificable).
            </div>
          </div>
        </div>

        <div style={styles.heroRight}>
          <div style={styles.imageFrame}>
            <img src="/mesa.jpg" alt="Mesa de nieve Saringa" style={styles.heroImage} />
          </div>
          <div style={styles.imageCaption}>
            Mesa de exhibición (ejemplo) 🍧
          </div>
        </div>
      </section>

      {/* Calculadora */}
      <section style={styles.section}>
        <h3 style={styles.sectionTitle}>🧮 Cotizador: Personas / Litros</h3>

        <div style={styles.modeToggleRow}>
          <button
            type="button"
            onClick={() => setMode("personas")}
            style={{
              ...styles.modeBtn,
              ...(mode === "personas" ? styles.modeBtnActive : null),
            }}
          >
            👥 Por personas
          </button>
          <button
            type="button"
            onClick={() => setMode("litros")}
            style={{
              ...styles.modeBtn,
              ...(mode === "litros" ? styles.modeBtnActive : null),
            }}
          >
            🧊 Por litros
          </button>
          <div style={styles.modeHint}>
            <span>🔁</span> Cambia el modo para hacer la cotización “viceversa”.
          </div>
        </div>

        <div style={styles.grid2}>
          {/* Slider personas */}
          <div style={styles.card}>
            <div style={styles.cardTitle}>Control deslizante</div>

            <label style={styles.label}>
              {mode === "personas" ? "Personas" : "Personas (estimadas)"}
            </label>

            <input
              type="range"
              min={Math.round(MIN_LITROS / 0.25)}
              max={Math.round(MAX_LITROS / 0.25)}
              step={1}
              value={people}
              onChange={(e) => setPeople(parseInt(e.target.value, 10))}
              style={styles.range}
              disabled={mode !== "personas"}
            />

            <div style={styles.sliderValueRow}>
              <div style={styles.bigNumber}>{people}</div>
              <div style={styles.smallText}>personas</div>
            </div>

            <div style={styles.ruleLine}>
              {infoReglas.texto}
            </div>

            <div style={styles.dualRow}>
              <div>
                <div style={styles.kpiLabel}>Litros estimados</div>
                <div style={styles.kpiValue}>{litrosRedondeados} L</div>
              </div>
              <div>
                <div style={styles.kpiLabel}>Precio por litro</div>
                <div style={styles.kpiValue}>${precioLitro} MXN/L</div>
              </div>
            </div>

            <div style={styles.totalBox}>
              <div style={styles.totalLabel}>💰 Total estimado</div>
              <div style={styles.totalValue}>${Math.round(subtotal)} MXN</div>
              <div style={styles.totalHint}>
                *El pago y el acuerdo final se hacen por teléfono/WhatsApp ({phone}).
              </div>
            </div>
          </div>

          {/* Slider litros */}
          <div style={styles.card}>
            <div style={styles.cardTitle}>Ajuste por litros</div>

            <label style={styles.label}>
              {mode === "litros" ? "Litros" : "Litros (estimados)"}
            </label>

            <input
              type="range"
              min={MIN_LITROS}
              max={MAX_LITROS}
              step={0.25} // 1 persona = 0.25 L con tu regla 5L/20pers
              value={litrosRedondeados}
              onChange={(e) => setLiters(parseFloat(e.target.value))}
              style={styles.range}
              disabled={mode !== "litros" || (applyFlavorRule && selectedFlavors.length > 0)}
            />

            <div style={styles.sliderValueRow}>
              <div style={styles.bigNumber}>{litrosRedondeados}</div>
              <div style={styles.smallText}>litros</div>
            </div>

            <div style={styles.ruleLine}>
              Reglas de precio: <b>20–25 L = $150</b>, <b>más de 25 L = $130</b> (mínimo $130/L).
            </div>

            <div style={styles.limitsLine}>
              Límites: <b>{MIN_LITROS}–{MAX_LITROS} L</b>
            </div>

            {(applyFlavorRule && selectedFlavors.length > 0) ? (
              <div style={styles.notice}>
                <span>🧾</span>
                <div>
                  Estás aplicando sabores: <b>{selectedFlavors.length} sabor(es)</b> → <b>{flavorsLiters} L</b>.
                  {flavorsWithinLimitText(flavorsDentroDeLimite)}
                </div>
              </div>
            ) : (
              <div style={styles.notice}>
                <span>🍓</span>
                <div>
                  Si eliges sabores abajo, cada sabor suma <b>5 litros</b> automáticamente (opcional).
                </div>
              </div>
            )}
          </div>
        </div>
      </section>

      {/* Sabores */}
      <section style={styles.section}>
        <h3 style={styles.sectionTitle}>🍧 Sabores (cada uno suma 5 litros)</h3>

        <div style={styles.card}>
          <div style={styles.rowBetween}>
            <div>
              <div style={styles.cardTitleSmall}>Selecciona</div>
              <div style={styles.subtleText}>
                Seleccionados: <b>{selectedFlavors.length}</b> → Litros por sabores: <b>{flavorsLiters} L</b>
              </div>
            </div>

            <label style={styles.checkboxRow}>
              <input
                type="checkbox"
                checked={applyFlavorRule}
                onChange={(e) => setApplyFlavorRule(e.target.checked)}
              />
              <span>Aplicar regla de sabores al cálculo</span>
            </label>
          </div>

          {!saboresDentroDeLimite && applyFlavorRule && selectedFlavors.length > 0 ? (
            <div style={{ ...styles.errorBox, marginTop: 10 }}>
              ⚠️ Tus sabores ya superan el máximo de <b>{MAX_LITROS} L</b>. Quita sabores para bajar a 150 L.
            </div>
          ) : null}

          <div style={styles.flavorsGrid}>
            {sabores.map((s) => {
              const active = selectedFlavors.includes(s);
              return (
                <button
                  key={s}
                  type="button"
                  onClick={() => toggleFlavor(s)}
                  style={{
                    ...styles.flavorChip,
                    ...(active ? styles.flavorChipActive : null),
                  }}
                  disabled={applyFlavorRule && !active && (flavorsLiters + 5) > MAX_LITROS}
                  title={(applyFlavorRule && flavorsLiters + 5 > MAX_LITROS) ? "Supera el máximo de 150 L" : ""}
                >
                  {s}
                </button>
              );
            })}
          </div>

          <div style={styles.smallDisclaimer}>
            *Los sabores disponibles son los listados. No se modifica el tamaño: <b>+5 L por sabor</b>.
          </div>
        </div>
      </section>

      {/* Confirmación y CTA */}
      <section style={styles.section}>
        <h3 style={styles.sectionTitle}>📦 Encargo y acuerdo</h3>

        <div className={styles.grid2}>
          <div style={styles.card}>
            <div style={styles.cardTitle}>Incluye</div>
            <ul style={styles.ul}>
              <li>📍 Lugar personalizado por el cliente en <b>Hidalgo</b></li>
              <li>⏱️ Servicio por <b>2 horas</b></li>
              <li>🥄 Material para pruebas: <b>vasos y cucharas</b></li>
              <li>🗓️ Solicitud con <b>1 semana</b> de anticipación</li>
            </ul>
            <div style={styles.subtleText}>
              Para cerrar fechas, disponibilidad y logística, siempre se contacta al número.
            </div>
          </div>

          <div style={styles.card}>
            <div style={styles.cardTitle}>Confirmar cotización</div>
            <div style={styles.summaryList}>
              <div className={styles.sumRow}>
                <span>👥 Personas</span>
                <b>{people}</b>
              </div>
              <div className={styles.sumRow}>
                <span>🧊 Litros</span>
                <b>{litrosRedondeados} L</b>
              </div>
              <div className={styles.sumRow}>
                <span>💲 Precio/L</span>
                <b>${precioLitro}</b>
              </div>
              <div className={styles.sumRow}>
                <span>💰 Estimado</span>
                <b>${Math.round(subtotal)} MXN</b>
              </div>
            </div>

            <div style={styles.ctaRow}>
              <a href={`tel:${phone}`} style={styles.primaryBtn}>
                📞 Llamar {phone}
              </a>
              <a href={whatsappLink} target="_blank" rel="noreferrer" style={styles.secondaryBtn}>
                💬 Enviar por WhatsApp
              </a>
            </div>

            <div style={styles.smallDisclaimer}>
              *Este sitio estima el costo. El pago y el acuerdo final se realizan únicamente al contactar el teléfono.
            </div>
          </div>
        </div>
      </section>

      {/* Footer */}
      <footer style={styles.footer}>
        <div style={styles.footerLeft}>
          <div style={styles.footerBrand}>Saringa 🍧🦋</div>
          <div style={styles.footerText}>
            Nieves 100% naturales, artesanales. Entregas y eventos en Hidalgo.
          </div>
        </div>
        <div style={styles.footerRight}>
          <div style={styles.footerContact}>📞 {phone}</div>
          <div style={styles.footerText}>Para pagos y dudas: por llamada o WhatsApp</div>
        </div>
      </footer>

      <style>{`
        .phoneLink { color: inherit; text-decoration: none; }
        .whatsBtn { display:inline-block; }
      `}</style>
    </div>
  );
}

function flavorsWithinLimitText(ok) {
  return ok ? (
    <span> ✅ Dentro del máximo.</span>
  ) : (
    <span> ⚠️ Supera el máximo.</span>
  );
}

const styles = {
  page: {
    minHeight: "100vh",
    background: "linear-gradient(180deg, #B8F2D5 0%, #CFF6E4 40%, #F8C9E6 120%)",
    color: "#0f0f0f",
    fontFamily: `"Trebuchet MS", "Comic Sans MS", Arial, sans-serif`,
    paddingBottom: 40,
  },
  header: {
    maxWidth: 1100,
    margin: "0 auto",
    padding: "18px 16px",
    display: "flex",
    alignItems: "flex-start",
    justifyContent: "space-between",
    gap: 16,
    flexWrap: "wrap",
  },
  brandWrap: {
    display: "flex",
    alignItems: "center",
    gap: 14,
  },
  logo: {
    width: 140,
    height: "auto",
    display: "block",
    borderRadius: 14,
    background: "rgba(255,255,255,0.55)",
    padding: 8,
    boxShadow: "0 10px 22px rgba(0,0,0,0.12)",
  },
  brandText: { display: "flex", flexDirection: "column" },
  brandName: {
    fontSize: 44,
    fontStyle: "italic",
    fontWeight: 900,
    letterSpacing: 0.2,
    color: "#111",
    lineHeight: 1,
  },
  brandSub: {
    marginTop: 6,
    fontSize: 16,
    fontWeight: 700,
    color: "#0e0e0e",
  },
  contactBox: {
    minWidth: 270,
    maxWidth: 360,
    borderRadius: 18,
    padding: 14,
    background: "rgba(255,255,255,0.62)",
    boxShadow: "0 10px 22px rgba(0,0,0,0.10)",
    border: "1px solid rgba(0,0,0,0.06)",
  },
  contactTitle: { fontWeight: 900, marginBottom: 8 },
  phoneLink: {
    display: "block",
    fontSize: 22,
    fontWeight: 900,
    color: "#111",
    marginBottom: 8,
  },
  whatsBtn: {
    display: "block",
    width: "100%",
    textAlign: "center",
    padding: "10px 12px",
    borderRadius: 12,
    background: "#ff4fa3",
    color: "#0b0b0b",
    fontWeight: 900,
    textDecoration: "none",
    border: "2px solid rgba(0,0,0,0.15)",
  },
  contactHint: {
    marginTop: 10,
    fontSize: 12,
    fontWeight: 700,
    color: "#1a1a1a",
  },

  hero: {
    maxWidth: 1100,
    margin: "0 auto",
    padding: "6px 16px 18px",
    display: "grid",
    gridTemplateColumns: "1.15fr 0.85fr",
    gap: 18,
    alignItems: "stretch",
  },
  heroLeft: {
    background: "rgba(255,255,255,0.65)",
    border: "1px solid rgba(0,0,0,0.06)",
    borderRadius: 20,
    padding: 18,
    boxShadow: "0 10px 22px rgba(0,0,0,0.10)",
  },
  heroTitle: { fontSize: 28, margin: "0 0 10px", fontWeight: 1000 },
  heroText: { margin: "8px 0", fontSize: 15, fontWeight: 750 },
  notice: {
    marginTop: 14,
    borderRadius: 16,
    padding: 12,
    background: "rgba(255,255,255,0.75)",
    border: "2px dashed rgba(0,0,0,0.10)",
    display: "flex",
    gap: 10,
    alignItems: "flex-start",
    fontWeight: 800,
  },

  heroRight: {
    background: "rgba(255,255,255,0.55)",
    borderRadius: 20,
    padding: 12,
    border: "1px solid rgba(0,0,0,0.06)",
    boxShadow: "0 10px 22px rgba(0,0,0,0.10)",
    display: "flex",
    flexDirection: "column",
    gap: 8,
  },
  imageFrame: {
    borderRadius: 16,
    overflow: "hidden",
    border: "1px solid rgba(0,0,0,0.08)",
    background: "#fff",
  },
  heroImage: {
    width: "100%",
    height: 320,
    objectFit: "cover",
    display: "block",
  },
  imageCaption: { textAlign: "center", fontWeight: 900 },

  section: {
    maxWidth: 1100,
    margin: "18px auto 0",
    padding: "0 16px",
  },
  sectionTitle: {
    fontSize: 22,
    fontWeight: 1000,
    margin: "14px 0 12px",
    color: "#0e0e0e",
  },

  grid2: {
    display: "grid",
    gridTemplateColumns: "1fr 1fr",
    gap: 14,
  },

  card: {
    background: "rgba(255,255,255,0.66)",
    border: "1px solid rgba(0,0,0,0.06)",
    borderRadius: 20,
    padding: 16,
    boxShadow: "0 10px 22px rgba(0,0,0,0.10)",
  },
  cardTitle: { fontWeight: 1000, fontSize: 16, marginBottom: 10 },
  cardTitleSmall: { fontWeight: 1000, marginBottom: 4 },
  label: { fontWeight: 900, fontSize: 14, display: "block", marginBottom: 6 },

  range: {
    width: "100%",
    accentColor: "#1aa884", // verde cerceta aproximado
    marginBottom: 10,
  },
  sliderValueRow: { display: "flex", gap: 10, alignItems: "baseline", marginBottom: 6 },
  bigNumber: { fontSize: 42, fontWeight: 1100, lineHeight: 1 },
  smallText: { fontSize: 14, fontWeight: 900 },

  ruleLine: {
    marginTop: 10,
    fontWeight: 800,
    fontSize: 12.5,
    color: "#222",
    background: "rgba(255,255,255,0.6)",
    padding: 10,
    borderRadius: 14,
    border: "1px solid rgba(0,0,0,0.06)",
  },

  dualRow: {
    display: "flex",
    gap: 12,
    marginTop: 12,
    flexWrap: "wrap",
  },
  kpiLabel: { fontWeight: 900, fontSize: 12 },
  kpiValue: { fontWeight: 1100, fontSize: 20 },
  totalBox: {
    marginTop: 14,
    borderRadius: 18,
    padding: 12,
    background: "linear-gradient(135deg, rgba(26,168,132,0.18) 0%, rgba(255,79,163,0.18) 100%)",
    border: "2px solid rgba(0,0,0,0.06)",
  },
  totalLabel: { fontWeight: 1000, fontSize: 14 },
  totalValue: { fontWeight: 1200, fontSize: 30 },
  totalHint: { marginTop: 6, fontSize: 12.5, fontWeight: 800 },

  limitsLine: { marginTop: 10, fontWeight: 900 },

  modeToggleRow: {
    display: "flex",
    alignItems: "center",
    gap: 10,
    flexWrap: "wrap",
    marginBottom: 14,
  },
  modeBtn: {
    padding: "10px 14px",
    borderRadius: 14,
    background: "rgba(255,255,255,0.55)",
    border: "1px solid rgba(0,0,0,0.10)",
    fontWeight: 1000,
    cursor: "pointer",
  },
  modeBtnActive: {
    background: "rgba(26,168,132,0.25)",
    border: "2px solid rgba(0,0,0,0.16)",
  },
  modeHint: { marginLeft: 6, fontWeight: 900, fontSize: 12 },

  rowBetween: { display: "flex", justifyContent: "space-between", gap: 10, alignItems: "center", flexWrap: "wrap" },
  checkboxRow: { display: "flex", gap: 10, alignItems: "center", fontWeight: 900, fontSize: 13 },

  flavorsGrid: {
    marginTop: 14,
    display: "grid",
    gridTemplateColumns: "repeat(3, minmax(0, 1fr))",
    gap: 10,
  },
  flavorChip: {
    padding: "10px 10px",
    borderRadius: 14,
    border: "1px solid rgba(0,0,0,0.12)",
    background: "rgba(255,255,255,0.55)",
    fontWeight: 1000,
    cursor: "pointer",
    textAlign: "center",
    fontSize: 12.5,
  },
  flavorChipActive: {
    background: "rgba(255,79,163,0.24)",
    border: "2px solid rgba(0,0,0,0.20)",
  },
  smallDisclaimer: { marginTop: 12, fontSize: 12.5, fontWeight: 850, color: "#1c1c1c" },

  errorBox: {
    borderRadius: 14,
    background: "rgba(255,79,163,0.16)",
    border: "2px solid rgba(0,0,0,0.12)",
    padding: 12,
    fontWeight: 1000,
  },

  ul: { margin: 0, paddingLeft: 18, fontWeight: 900, lineHeight: 1.7 },
  summaryList: { marginTop: 8 },
  sumRow: { display: "flex", justifyContent: "space-between", padding: "8px 0", borderBottom: "1px dashed rgba(0,0,0,0.12)", fontWeight: 950 },
  ctaRow: { display: "grid", gridTemplateColumns: "1fr 1fr", gap: 10, marginTop: 14 },
  primaryBtn: {
    textDecoration: "none",
    background: "#1aa884",
    color: "#0a0a0a",
    border: "2px solid rgba(0,0,0,0.16)",
    borderRadius: 14,
    padding: "12px 10px",
    textAlign: "center",
    fontWeight: 1100,
  },
  secondaryBtn: {
    textDecoration: "none",
    background: "#ff4fa3",
    color: "#0a0a0a",
    border: "2px solid rgba(0,0,0,0.16)",
    borderRadius: 14,
    padding: "12px 10px",
    textAlign: "center",
    fontWeight: 1100,
  },

  smallText2: { fontSize: 12 },
  subtleText: { marginTop: 6, fontSize: 13, fontWeight: 850, color: "#1b1b1b" },

  footer: {
    maxWidth: 1100,
    margin: "26px auto 0",
    padding: "0 16px 18px",
    display: "flex",
    justifyContent: "space-between",
    gap: 12,
    flexWrap: "wrap",
    color: "#101010",
  },
  footerLeft: {
    background: "rgba(255,255,255,0.55)",
    border: "1px solid rgba(0,0,0,0.06)",
    borderRadius: 18,
    padding: 14,
    boxShadow: "0 10px 22px rgba(0,0,0,0.08)",
    minWidth: 300,
  },
  footerRight: {
    background: "rgba(255,255,255,0.55)",
    border: "1px solid rgba(0,0,0,0.06)",
    borderRadius: 18,
    padding: 14,
    boxShadow: "0 10px 22px rgba(0,0,0,0.08)",
    minWidth: 260,
  },
  footerBrand: { fontWeight: 1200, fontStyle: "italic", fontSize: 22 },
  footerText: { marginTop: 6, fontWeight: 850, fontSize: 13.5, lineHeight: 1.6 },
  footerContact: { fontWeight: 1200, fontSize: 18 },
};
