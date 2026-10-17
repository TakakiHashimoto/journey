---
type: concept
---

# Basic Calender logic:

```javascript

const firstDayOfMonth = new Date(displayYear, displayMonth, 1) 
// "first-day-of-the-month" object 
const daysInMonth = new Date(displayYear, displayMonth + 1, 0).getDate() 
// 0 means last day of previous month. So, if you displayMonth + 1, 0 => It gives the last day of this month 
// => = how many days there are in this month 
const startDay = firstDayOfMonth.getDay() 
// What day is the first day of the month 
const startOffset = startDay === 0 ? 6 : startDay - 1 
const celenderSchedeules: calenderCell[] = [ ...Array.from({ length: startOffset }, () => ({ kind: "empty" as const })), ...Array.from({ length: daysInMonth }, (_, i) => { const day = i + 1 
const month = String(displayMonth + 1).padStart(2, "0") 
const date = String(day).padStart(2, "0") 
return { kind: "day" as const, day: day, date: ${displayYear}-${month}-${date}, } }), ]

```

