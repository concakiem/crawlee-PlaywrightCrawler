# crawlee-PlaywrightCrawler
crawlee sử dụng hàm PlaywrightCrawler

import { PlaywrightCrawler, Dataset } from 'crawlee';
import fs from 'fs';

const startUrls = ['https://teespring.com/shop/dogs?utm_campaign=marketplace&utm_medium=website&utm_source=teespring'];
const MAX_DEPTH = 2; // Giới hạn 2 tầng

const crawler = new PlaywrightCrawler({
    async requestHandler({ page, request, enqueueLinks, log }) {
        const { depth = 0 } = request.userData; // Lấy depth từ request
        log.info(`Crawling [depth=${depth}]: ${request.url}`);

        // Lấy tiêu đề trang
        const title = await page.title();

        // Lấy tất cả link trong trang
        const links = await page.$$eval('a', anchors =>
            anchors.map(a => a.href).filter(Boolean)
        );

        // Lưu dữ liệu
        await Dataset.pushData({
            url: request.url,
            depth,
            title,
            links,
        });

        // Nếu chưa đạt depth tối đa → enqueue link con
        if (depth < MAX_DEPTH) {
            await enqueueLinks({
                selector: 'a',
                baseUrl: request.loadedUrl,
                userData: { depth: depth + 1 }, // Tăng depth khi thêm request mới
            });
        }
    },

    maxRequestsPerCrawl: 50, // tránh crawl quá nhiều
    headless: true, // chạy không mở cửa sổ browser
});

// Chạy crawler
await crawler.run(startUrls.map(url => ({ url, userData: { depth: 0 } })));

// Xuất dữ liệu ra file sau khi crawl
const dataset = await Dataset.open();
const data = await dataset.getData();

// JSON
fs.writeFileSync('output.json', JSON.stringify(data.items, null, 2));

// CSV
if (data.items.length > 0) {
    const headers = Object.keys(data.items[0]).join(',');
    const csv = [
        headers,
        ...data.items.map(obj => Object.values(obj).map(v => `"${v}"`).join(','))
    ].join('\n');
    fs.writeFileSync('output.csv', csv);
}

console.log(`✅ Crawl hoàn tất! Depth <= ${MAX_DEPTH}. Kết quả lưu tại output.json và output.csv`);
