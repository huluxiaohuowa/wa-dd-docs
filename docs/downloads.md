# WA-DD 下载

<div class="wa-downloads" id="wa-downloads">
  <p class="wa-downloads-status">正在读取最新发布版本...</p>
  <div class="wa-downloads-grid">
    <a class="wa-download-card" id="wa-vos-package" href="https://github.com/huluxiaohuowa/wa-dd-docs/releases/latest" target="_blank" rel="noopener">
      <span class="wa-download-title">VOS 应用安装包</span>
      <span class="wa-download-name">等待最新版本</span>
      <span class="wa-download-meta">用于 ictrek.app / VOS 应用商店导入安装。</span>
    </a>
    <a class="wa-download-card" id="wa-deploy-package" href="https://github.com/huluxiaohuowa/wa-dd-docs/releases/latest" target="_blank" rel="noopener">
      <span class="wa-download-title">独立部署包</span>
      <span class="wa-download-name">等待最新版本</span>
      <span class="wa-download-meta">用于服务器上直接替换或启动 WA-DD 部署目录。</span>
    </a>
  </div>
  <p class="wa-downloads-fallback">
    如果自动读取失败，可以打开
    <a href="https://github.com/huluxiaohuowa/wa-dd-docs/releases/latest" target="_blank" rel="noopener">最新发布页</a>
    手动下载。
  </p>
</div>

<script>
(function () {
  var repo = "huluxiaohuowa/wa-dd-docs";
  var latestReleaseUrl = "https://github.com/" + repo + "/releases/latest";
  var status = document.querySelector("#wa-downloads .wa-downloads-status");
  var vos = document.getElementById("wa-vos-package");
  var deploy = document.getElementById("wa-deploy-package");

  function setCard(card, asset) {
    card.href = asset.browser_download_url;
    card.querySelector(".wa-download-name").textContent = asset.name;
  }

  function findAsset(assets, pattern) {
    return assets.find(function (asset) {
      return pattern.test(asset.name);
    });
  }

  fetch("https://api.github.com/repos/" + repo + "/releases/latest", {
    headers: { Accept: "application/vnd.github+json" }
  })
    .then(function (response) {
      if (!response.ok) {
        throw new Error("release request failed: " + response.status);
      }
      return response.json();
    })
    .then(function (release) {
      var assets = release.assets || [];
      var vosAsset = findAsset(assets, /^wa-dd_[0-9]+\.[0-9]+\.[0-9]+_pull\.tar$/);
      var deployAsset = findAsset(assets, /^wa-dd_deploy_[0-9]+\.[0-9]+\.[0-9]+\.tar\.gz$/);
      if (!vosAsset || !deployAsset) {
        throw new Error("expected package assets were not found");
      }
      setCard(vos, vosAsset);
      setCard(deploy, deployAsset);
      status.textContent = "当前最新版本：" + release.tag_name;
    })
    .catch(function () {
      status.textContent = "暂时无法自动读取最新发布版本。";
      vos.href = latestReleaseUrl;
      deploy.href = latestReleaseUrl;
    });
})();
</script>

<style>
.wa-downloads {
  margin-top: 1.25rem;
}
.wa-downloads-status {
  color: var(--md-default-fg-color--light);
}
.wa-downloads-grid {
  display: grid;
  grid-template-columns: repeat(auto-fit, minmax(240px, 1fr));
  gap: 1rem;
  margin: 1rem 0;
}
.wa-download-card {
  display: grid;
  gap: 0.45rem;
  padding: 1rem;
  border: 1px solid var(--md-default-fg-color--lightest);
  border-radius: 0.4rem;
  color: var(--md-default-fg-color);
  text-decoration: none;
}
.wa-download-card:hover {
  border-color: var(--md-accent-fg-color);
}
.wa-download-title {
  font-weight: 700;
}
.wa-download-name {
  color: var(--md-accent-fg-color);
  overflow-wrap: anywhere;
}
.wa-download-meta,
.wa-downloads-fallback {
  color: var(--md-default-fg-color--light);
}
</style>
