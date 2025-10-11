# up to date:

- e.g:
- (GET)
- https://footlordindustries.altervista.org/download_datapack_v2.php?name=datapack-september-2025&date=24-09-2025&version=167
> https://footlordindustries.altervista.org/download_datapack_v2.php?name=DATA_PACK_NAME&date=DATE_FORMAT&version=GAME_VER

- get name & version
- GET NAME LIST
```js
const axios = require("axios");

async function getDatapackList() {
  try {
    const url = "https://footlordindustries.altervista.org/get_datapack_list.php";

    // biasanya PHP endpoint ini terima POST kosong (body bisa kosong)
    const res = await axios.post(url);

    console.log("Status:", res.status);
    console.log("Data:", res.data); // kemungkinan JSON atau array
  } catch (err) {
    console.error("Request error:", err.message);
  }
}

getDatapackList();
```

- GET VERSION (GET)
- https://footlordindustries.altervista.org/checkVersion.php?version=167
