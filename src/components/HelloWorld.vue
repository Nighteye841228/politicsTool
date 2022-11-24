<template>
  <div class="container grid grid-cols-5 justify-center gap-4 font-serif">
    <img
      class="col-span-3 col-start-2 justify-self-center"
      alt="Vue logo"
      src="../assets/logo.png"
    />
    <h1 class="col-span-1 col-start-3 text-xl">{{ msg }}</h1>

    <a
      href="https://docusky.org.tw/DocuSky/docuTools/userMain/"
      class="col-span-1 col-start-3 bg-red-50 text-blue-500 hover:bg-red-100 text-center"
      target="_blank"
      >DocuSky官網</a
    >
    <span class="bg-gray-50 col-start-2 col-span-1 py-2">帳號</span>
    <input
      type="text"
      v-model="account"
      class="bg-green-50 col-start-3 col-span-2 py-2"
    />
    <span class="bg-gray-50 col-start-2 col-span-1 py-2">密碼</span>
    <input
      type="password"
      class="bg-green-50 col-start-3 col-span-2 py-2"
      v-model="password"
    />
    <button
      class="col-span-3 col-start-2 bg-red-50 hover:bg-red-100 rounded-md py-2"
      @click="login"
    >
      登入
    </button>
    <span class="bg-gray-50 col-start-2 col-span-1 py-2">文獻集名稱</span>
    <input
      class="bg-green-50 col-start-3 col-span-2 hover:bg-green-100 py-2"
      type="text"
      v-model="corpus"
      :placeholder="'自訂名稱'"
    />

    <input
      class="bg-green-50 col-start-2 col-span-3"
      type="file"
      ref="file"
      @change="readFile()"
    />
    <button
      class="col-span-3 col-start-2 bg-red-50 hover:bg-red-100 rounded-md py-2"
      @click="uploadXML"
    >
      上傳
    </button>
    <textarea
      rows="50"
      class="col-span-3 col-start-2 bg-green-50 hover:bg-green-100"
      placeholder="可直接在此貼上csv原始碼，或將從政府檔案庫收藏下載的檔案上傳"
      v-model="sample"
    ></textarea>
  </div>
</template>

<style scoped></style>

<script setup>
import { ref, computed } from "vue";
import { DataFrame, toJSON as ts } from "danfojs";
import { create } from "xmlbuilder2";
import csv from "csvtojson";

/* global defineProps */
defineProps({
  msg: String,
});

const sample = ref("");
const finalXML = ref("");
const account = ref("");
const password = ref("");
const waitToTrans = ref("");
const corpus = ref("");
const transJSON = ref({});

const file = ref(null);
const readFile = () => {
  let csvString = file.value.files[0];
  const reader = new FileReader();
  reader.onload = async (res) => {
    let temp = res.target.result;
    sample.value = temp;
  };
  reader.onerror = (err) => console.log(err);
  reader.readAsText(csvString);
};

let meta_dic = {
  題名: "title",
  刊物: "compilation_name",
  資料來源: "doc_source",
  文獻類型: "doctype",
  出版日期新: "time_org_str",
  發文字號: "publish_vol",
  摘要: "doc_content",
  附錄: "appendix",
  備註: "remark",
  相關人員: "related_person",
  相關組織: "related_org",
  相關法規: "related_law",
  "議長/省長/院長": "chairperson",
  在位總統: "president",
  引用格式: "quote_format",
  會議開始日期: "timeseq_not_before",
  會議結束日期: "timeseq_not_after",
  類別新: "docclass",
  卷期頁次: "compilation_vol",
  會議開始日期_AD: "year_for_grouping",
};

const timeNow = new Date();
const corpusName = computed(() => {
  return String(`${corpus.value}_${timeNow.toLocaleString()}`).replace(
    /[/\s]/g,
    "_"
  );
});

async function csvTrans() {
  //console.log("1");
  let waitToTransCsvString = sample.value
    .replace(/^.*\n.*\n.*\n/g, "")
    .replace(/"\t*"/g, "")
    .replace(/\t/g, ",")
    .replace(/=/g, "");

  waitToTrans.value = await csv().fromString(waitToTransCsvString);
  await transData();
}

async function transData() {
  let df = new DataFrame(waitToTrans.value);

  df.fillNa('""');
  //console.log(df);
  // # 資料來源、文獻類型合併
  // df['刊物'] = df['資料來源'] + df['文獻類型']
  df.addColumn("刊物", df["資料來源"].str.concat(df["文獻類型"].values, 1), {
    inplace: true,
  });
  //console.log("wwwwwwwwwww");
  // // // # 類別、主題類別合併
  // df['類別'] = df['類別'] + ';' + df['主題分類']
  let sss = df["主題分類"].map((x) => {
    return x === "" ? "" : ";" + x;
  });

  df.addColumn("類別新", df["類別"].str.concat(sss.values, 1), {
    inplace: true,
  });

  // df['卷期頁次'] = df["卷期"] + df["頁次"]
  df.addColumn(
    "卷期頁次",
    df["卷期"].str.join("", ";").str.concat(df["頁次"].values, 1),
    {
      inplace: true,
    }
  );

  // df['出版日期'] = df['出版日期'].map(trans_minguo)

  // // # 找到會議的開始跟結束
  // df['會議開始日期'] = df['會議日期'].map(meeting_start_time)
  // df['會議結束日期'] = df['會議日期'].map(meeting_end_time)

  // df['會議開始日期_AD'] = df['會議開始日期'].map(x => x.replace(/(\d+)\/(\d+)\/(\d+)/g, '$1'))
  // df['出版日期'] = df['出版日期'].map(x => x.replace(/\//g, ''))
  // df['會議開始日期'] = df['會議開始日期'].map(x => x.replace(/\//g, ''))

  df.addColumn(
    "出版日期新",
    df["出版日期"].map((x) => {
      let transChiToAD = String(x);
      transChiToAD = transChiToAD.replace(/民國(\d+)年.*/g, "$1");
      transChiToAD = String(Number(transChiToAD) + 1911);
      x = x.replace(/民國(\d+)年/, transChiToAD);
      x = x.replace(/[月日]/g, "");
      return String(x).replace(/\//g, "");
    }),
    { inplace: true }
  );

  df.addColumn(
    "會議開始日期",
    df["會議日期"].map((x) => {
      if (!x.match(/\d/)) return "99990000";

      if (x.match(/~/)) {
        x = x.replace(/(^[^~]*).*/, "$1").trim();
      }
      return String(x).replace(/\//g, "");
    }),
    { inplace: true }
  );

  df.addColumn(
    "會議開始日期_AD",
    df["會議日期"].map((x, ind) => {
      if (!x.match(/\d/)) {
        if (df["出版日期新"].getColumnData[ind].match(/\d/)) {
          x = String(df["出版日期新"].getColumnData[ind]);
        } else {
          return "";
        }
      }

      if (x.match(/~/)) {
        x = x.replace(/(^[^~]*).*/, "$1").trim();
      }
      return String(x).replace(/(\d{4}).*/g, "$1");
    }),
    { inplace: true }
  );

  df.addColumn(
    "會議結束日期",
    df["會議日期"].map((x) => {
      if (!x.match(/\d/)) return "99990000";

      if (x.match(/~/)) {
        //console.log("BE", x);
        x = x.replace(/(.*)~/g, "").trim();
        //console.log("AF", x);
      }
      return String(x).replace(/\//g, "");
    }),
    { inplace: true }
  );

  df = df.rename(meta_dic);
  df.drop({
    columns: [
      "no",
      "類別",
      "主題分類",
      "會議屆次",
      "會議日期",
      "出版日期",
      "卷期",
      "頁次",
    ],
    inplace: true,
  });
  transJSON.value = ts(df);

  await outXML();
}

async function outXML() {
  let politicJSON = transJSON.value;
  let fileNum = Math.ceil(Math.log10(politicJSON.length));
  // console.log(fileNum)

  let root = create({ version: "1.0" }).ele("ThdlPrototypeExport");
  let corpus = root.ele("corpus", {
    name: corpusName.value,
  });
  let metadata_field_settings = corpus.ele("metadata_field_settings");
  for (let key in meta_dic) {
    metadata_field_settings
      .ele(meta_dic[key], { show_spotlight: "Y", display_order: "999" })
      .txt(key);
  }
  corpus
    .ele("feature_analysis")
    .ele("spotlight", {
      category: "Udef_article_category",
      sub_category: "-",
      display_order: "1",
      title: "種類",
    })
    .up()
    .ele("tag", {
      type: "contentTagging",
      name: "Udef_article_category",
      default_category: "Udef_article_category",
      default_sub_category: "-",
    });

  // console.log(root.end({ prettyPrint: true }));
  let documents = root.ele("documents");
  let doc_content = undefined;
  politicJSON.forEach((article, index) => {
    let document = documents.ele("document", {
      filename: `${padding(index, fileNum)}`,
    });
    document.ele("corpus").txt(corpusName.value);
    for (let key in article) {
      if (key === "docclass" || key === "quote_format") {
        continue;
      } else if (key === "doc_content") {
        doc_content = document.ele(key).txt(article[key]);
      } else {
        document.ele(key).txt(article[key]);
      }
    }

    let xml_metadata = document.ele("xml_metadata");
    for (let key in article) {
      if (key !== "doc_content") {
        xml_metadata.ele(key).txt(article[key]);
      }
    }

    let metatag = doc_content.ele("MetaTags");
    article["docclass"].split(";").forEach((x) => {
      metatag.ele("Udef_article_category").txt(x);
    });
    doc_content
      .ele("Comment")
      .ele("CommentItem", { Category: "引用格式" })
      .txt(article["quote_format"]);
  });

  console.log(root.end({ prettyPrint: true }));
  finalXML.value = root.end({ prettyPrint: true });
}

function login() {
  if (!account.value || !password.value) {
    fail();
    return;
  }
  // eslint-disable-next-line
  docuskyManageDbListSimpleUI.login(
    account.value,
    password.value,
    () => {
      alert("登入成功");
    },
    function () {
      alert("nonono");
    }
  );
  function fail() {
    alert("failed");
  }
}

async function uploadXML() {
  await csvTrans();
  let formData = {
    dummy: {
      name: "dbTitleForImport",
      value: corpusName.value,
    },
    file: {
      value: finalXML.value,
      filename: "corpusName" + ".xml",
      name: "importedFiles[]",
    },
  };
  //console.log("4");
  // eslint-disable-next-line
  docuskyManageDbListSimpleUI.uploadMultipart(
    formData,
    await function () {
      alert("上傳成功!");
    },
    await function () {
      alert("失敗，請重試或檢查是否檔案錯誤");
    }
  );
}

function padding(num, length) {
  for (var len = (num + "").length; len < length; len = num.length) {
    num = "0" + num;
  }
  return num;
}
</script>
