
---

✅ 1. শেষ ২০ মিনিটে কে কে কানেক্ট করেছে (ইউজার + আইপি)

```bash
awk -v d1="$(date --date='20 minutes ago' '+%Y-%m-%d %H:%M:%S')" \
    -v d2="$(date '+%Y-%m-%d %H:%M:%S')" \
    '$1 >= d1 && $1 <= d2' /var/log/xray/access.log | awk '{print $3, $5}' | sort | uniq
```

উদাহরণ আউটপুট:
```
user1@example.com 103.123.45.67
user2@example.com 45.98.12.5
```


---

✅ 2. মাল্টিপল আইপি দিয়ে লগইন করা ইউজারদের লিস্ট

```bash
awk '{print $3, $5}' /var/log/xray/access.log | sort | uniq | \
awk '{count[$1]++} END {for (u in count) if (count[u]>1) print u, count[u]}' | sort -k2 -nr
```

উদাহরণ আউটপুট:
```
user1@example.com 5
user3@example.com 2
```


---

✅ 3. প্রতিটি ইউজারের আইপি লিস্ট সহ রিপোর্ট

```bash
awk '{print $3, $5}' /var/log/xray/access.log | sort | uniq | \
awk '{ips[$1]=ips[$1]","$2} END {for (u in ips) print u, ips[u]}'
```

উদাহরণ আউটপুট:
```
user1@example.com ,103.123.45.67,103.123.46.22,185.12.55.33
user2@example.com ,45.98.12.5
```


---

✅ 4. Live কানেক্টেড IP (443, 10000 পোর্টে)

```bash
ss -tnp | grep -E ':443|:10000' | grep ESTAB
```

উদাহরণ আউটপুট:
```
ESTAB 0 0 103.123.45.67:443 111.222.33.44:59120 users:(("xray",pid=1234,fd=9))
ESTAB 0 0 103.123.45.67:10000 223.234.56.78:49832 users:(("xray",pid=1234,fd=10))
```


---

✅ 5. Real-time access log দেখা

```bash
tail -f /var/log/xray/access.log
```

উদাহরণ আউটপুট:
```
2025-05-26 15:10:02 user1@example.com 103.123.45.67 accepted tcp:google.com:443
2025-05-26 15:10:05 user2@example.com 45.98.12.5 accepted tcp:telegram.org:443
```


---

✅ 6. নির্দিষ্ট ইউজার কবে লগইন করেছে (timestamp সহ)

```bash
grep 'user1@example.com' /var/log/xray/access.log
```

উদাহরণ আউটপুট:
```
2025-05-26 14:55:31 user1@example.com 103.123.45.67 accepted tcp:example.com:443
2025-05-26 15:10:02 user1@example.com 185.12.55.33 accepted tcp:google.com:443
```


---

