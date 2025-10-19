# CDN File for Access
> https://cdn.jsdelivr.net/gh/pren8/footballdatabase/data2/datapack-september-2025_25-09-2025_156.json?v=2

# Download New Datapack
> https://footlordindustries.altervista.org/download_datapack_v2.php?name=datapack-september-2025&date=24-09-2025&version=167 (GET)
---
> https://footlordindustries.altervista.org/download_datapack_v2.php?name=DATA_PACK_NAME&date=DATE_FORMAT&version=GAME_VER (GET)

# Get Data Pack List
> https://footlordindustries.altervista.org/get_datapack_list.php (GET)
```js
const axios = require("axios");
async function getDatapackList() {
  try {
    const url = "https://footlordindustries.altervista.org/get_datapack_list.php";
    const res = await axios.post(url);
    console.log("Status:", res.status);
    console.log("Data:", res.data);
  } catch (err) {
    console.error("Request error:", err.message);
  }
}
getDatapackList();
```

# Get Version Info
> https://footlordindustries.altervista.org/checkVersion.php?version=167 (GET)
