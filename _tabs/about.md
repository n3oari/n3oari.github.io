---
layout: page
title: About me
icon: fas fa-info-circle
order: 4
---

<div class="intro-hero">
  <p class="intro-tagline">Ciberseguridad, Red Teaming y más. 🇪🇸/🇬🇧</p>
  <br>
  <p class="intro-quote">~ Look like an admin, not an attacker. ~</p>
  <img src="/assets/img/ghost.gif" alt="ghost" class="intro-ghost">
</div>

<style>
.intro-hero {
  display: flex;
  flex-direction: column;
  align-items: center;
  text-align: center;
  gap: 0.6rem;
  margin: 1.5rem 0 2rem;
}

.intro-ghost {
  width: 560px;
  max-width: 100%;
  height: auto;
  filter: drop-shadow(0 0 16px rgba(0, 200, 120, 0.35));
}

.intro-tagline {
  width: 100%;
  text-align: left;
  font-size: 1.05rem;
  font-weight: 500;
  margin: 0;
}

.intro-quote {
  font-family: 'Courier New', Consolas, monospace;
  font-size: 0.9rem;
  letter-spacing: 0.03em;
  color: #2e7d32;
  border-top: 1px dashed rgba(46, 125, 50, 0.4);
  border-bottom: 1px dashed rgba(46, 125, 50, 0.4);
  padding: 0.4rem 1rem;
  margin: 0.3rem 0 0;
}
</style>

---

### Certificaciones Ciberseguridad


<div class="cert-carousel" id="certCarousel">
  <div class="cert-track">
    <div class="cert-slide">
      <img src="/assets/img/CRTP_CERT.jpg" alt="CRTP">
      <div class="cert-caption">
        <span class="cert-name">CRTP</span>
      </div>
    </div>
    <div class="cert-slide">
      <img src="/assets/img/bscp.jpeg" alt="BSCP">
      <div class="cert-caption">
        <span class="cert-name">BSCP</span>
      </div>
    </div>
    <div class="cert-slide">
      <img src="/assets/img/ejpt.jpg" alt="eJPT">
      <div class="cert-caption">
        <span class="cert-name">eJPT</span>
      </div>
    </div>
  </div>

  <button class="cert-btn cert-prev" aria-label="Anterior">&#10094;</button>
  <button class="cert-btn cert-next" aria-label="Siguiente">&#10095;</button>

  <div class="cert-dots"></div>
</div>

<style>
.cert-carousel {
  position: relative;
  max-width: 720px;
  margin: 1.5rem auto;
  overflow: hidden;
  border-radius: 12px;
  background: var(--card-bg, rgba(0,0,0,0.03));
  padding: 2rem 0 1.5rem;
}

.cert-track {
  display: flex;
  transition: transform 0.4s ease;
}

.cert-slide {
  flex: 0 0 100%;
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 1rem;
}

.cert-slide img {
  height: 380px;
  max-width: 90%;
  width: auto;
  object-fit: contain;
  border-radius: 8px;
}

.cert-caption {
  display: flex;
  flex-direction: column;
  align-items: center;
  gap: 0.35rem;
}

.cert-name {
  font-weight: 600;
  font-size: 1.1rem;
}

.cert-badge {
  font-size: 0.75rem;
  padding: 0.15rem 0.6rem;
  border-radius: 999px;
  color: #fff;
}

.cert-actual { background: #2e7d32; }
.cert-progreso { background: #ef6c00; }
.cert-futura { background: #757575; }

.cert-btn {
  position: absolute;
  top: 50%;
  transform: translateY(-50%);
  background: rgba(0,0,0,0.35);
  color: #fff;
  border: none;
  width: 42px;
  height: 42px;
  border-radius: 50%;
  cursor: pointer;
  font-size: 1.2rem;
  line-height: 1;
}

.cert-btn:hover { background: rgba(0,0,0,0.55); }
.cert-prev { left: 12px; }
.cert-next { right: 12px; }

.cert-dots {
  display: flex;
  justify-content: center;
  gap: 0.4rem;
  margin-top: 1rem;
}

.cert-dot {
  width: 9px;
  height: 9px;
  border-radius: 50%;
  background: rgba(128,128,128,0.4);
  cursor: pointer;
  border: none;
  padding: 0;
}

.cert-dot.active { background: rgba(128,128,128,1); }
</style>

<script>
(function () {
  const carousel = document.getElementById('certCarousel');
  const track = carousel.querySelector('.cert-track');
  const slides = carousel.querySelectorAll('.cert-slide');
  const dotsWrap = carousel.querySelector('.cert-dots');
  let index = 0;

  slides.forEach((_, i) => {
    const dot = document.createElement('button');
    dot.className = 'cert-dot' + (i === 0 ? ' active' : '');
    dot.addEventListener('click', () => goTo(i));
    dotsWrap.appendChild(dot);
  });
  const dots = dotsWrap.querySelectorAll('.cert-dot');

  function goTo(i) {
    index = (i + slides.length) % slides.length;
    track.style.transform = `translateX(-${index * 100}%)`;
    dots.forEach((d, di) => d.classList.toggle('active', di === index));
  }

  carousel.querySelector('.cert-prev').addEventListener('click', () => goTo(index - 1));
  carousel.querySelector('.cert-next').addEventListener('click', () => goTo(index + 1));

  let timer = setInterval(() => goTo(index + 1), 4000);
  carousel.addEventListener('mouseenter', () => clearInterval(timer));
  carousel.addEventListener('mouseleave', () => { timer = setInterval(() => goTo(index + 1), 4000); });
})();
</script>

### Otras Certificaciones

<div class="cert-carousel" id="certCarousel2">
  <div class="cert-track">
    <div class="cert-slide">
      <img src="/assets/img/Certiport%20IT%20Specialist%20-%20Databases-1.png" alt="Database Specialist">
      <div class="cert-caption">
        <span class="cert-name">Database Specialist</span>
      </div>
    </div>
    <div class="cert-slide">
      <img src="/assets/img/Certiport%20IT%20Specialist%20-%20Java-1.png" alt="Java Specialist">
      <div class="cert-caption">
        <span class="cert-name">Java Specialist</span>
      </div>
    </div>
    <div class="cert-slide">
      <img src="/assets/img/Cisco%20Linux%20fundamentals-1.png" alt="Linux Fundamentals">
      <div class="cert-caption">
        <span class="cert-name">Linux Fundamentals</span>
      </div>
    </div>
    <div class="cert-slide">
      <img src="/assets/img/Scrum-1.png" alt="Scrum">
      <div class="cert-caption">
        <span class="cert-name">Scrum</span>
      </div>
    </div>
  </div>

  <button class="cert-btn cert-prev" aria-label="Anterior">&#10094;</button>
  <button class="cert-btn cert-next" aria-label="Siguiente">&#10095;</button>

  <div class="cert-dots"></div>
</div>

<script>
(function () {
  const carousel = document.getElementById('certCarousel2');
  const track = carousel.querySelector('.cert-track');
  const slides = carousel.querySelectorAll('.cert-slide');
  const dotsWrap = carousel.querySelector('.cert-dots');
  let index = 0;

  slides.forEach((_, i) => {
    const dot = document.createElement('button');
    dot.className = 'cert-dot' + (i === 0 ? ' active' : '');
    dot.addEventListener('click', () => goTo(i));
    dotsWrap.appendChild(dot);
  });
  const dots = dotsWrap.querySelectorAll('.cert-dot');

  function goTo(i) {
    index = (i + slides.length) % slides.length;
    track.style.transform = `translateX(-${index * 100}%)`;
    dots.forEach((d, di) => d.classList.toggle('active', di === index));
  }

  carousel.querySelector('.cert-prev').addEventListener('click', () => goTo(index - 1));
  carousel.querySelector('.cert-next').addEventListener('click', () => goTo(index + 1));

  let timer = setInterval(() => goTo(index + 1), 4000);
  carousel.addEventListener('mouseenter', () => clearInterval(timer));
  carousel.addEventListener('mouseleave', () => { timer = setInterval(() => goTo(index + 1), 4000); });
})();
</script>
