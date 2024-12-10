---
title: 友情链接
date: 2024-12-09 21:37:32
type: "links"
---

<style>
.links-container {
  display: grid;
  grid-template-columns: repeat(auto-fill, minmax(250px, 1fr));
  gap: 20px;
  margin-top: 40px;
}

.link-item {
  background-color: #fff;
  border-radius: 8px;
  box-shadow: 0 4px 8px rgba(0, 0, 0, 0.1);
  transition: transform 0.3s ease, box-shadow 0.3s ease;
}

.link-item:hover {
  transform: translateY(-5px);
  box-shadow: 0 8px 16px rgba(0, 0, 0, 0.2);
}

.link-card {
  display: flex;
  flex-direction: column;
  justify-content: center;
  align-items: center;
  padding: 20px;
  text-decoration: none;
  color: #333;
  border-radius: 8px;
}

.link-card:hover {
  background-color: #f5f5f5;
}

.link-icon img {
  width: 80px;
  height: 80px;
  margin-bottom: 15px;
  border-radius: 50%;
  object-fit: cover;
}

.link-info h3 {
  font-size: 18px;
  font-weight: bold;
  margin-bottom: 5px;
}

.link-info p {
  font-size: 14px;
  color: #666;
  text-align: center;
  line-height: 1.5;
}

/* 增加响应式设计 */
@media (max-width: 768px) {
  .links-container {
    grid-template-columns: 1fr;
  }
}
</style>

<div class="links-container">
  <div class="link-item">
    <a href="https://linux.do/?source=moralok_com" target="_blank" class="link-card">
      <div class="link-icon">
        <img src="/images/linux_do.png" alt="LINUX DO" />
      </div>
      <div class="link-info">
        <h3>LINUX DO</h3>
        <p>新的理想型社区</p>
      </div>
    </a>
  </div>
</div>

