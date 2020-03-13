---
title: 实现date类
date: 2020-03-01 00:00:00
categories: C/C++
tags: date类
---

```cpp
// date.h
#pragma once

#include <iostream>

using std::cin;
using std::cout;
using std::endl;
using std::istream;
using std::ostream;

class Date {
    friend ostream& operator<<(ostream& _cout, const Date& d);
    friend istream& operator>>(istream& _cin, Date& d);
public:
    inline int GetMonthDay(int year, int month) const {
        static int monthArray[13] = { 0, 31, 28, 31, 30, 31, 30, 31, 31, 30, 31, 30, 31 };
        if ((month == 2)
            && ((year % 4 == 0 && year % 100 != 0)
                || (year % 400 == 0))) {
            return 29;
        }
        return monthArray[month];
    }

    Date(int year = 1900, int month = 1, int day = 1);
    int operator-(const Date&d) const;
    Date& operator++();		// 前置++
    Date operator++(int);	// 后置++
    Date& operator--();		// 前置--
    Date operator--(int);	// 后置--
    Date operator+(int day) const;
    Date operator-(int day) const;
    Date& operator+=(int day);
    Date& operator-=(int day);
    bool operator>(const Date& d) const;
    bool operator>=(const Date& d) const;
    bool operator<(const Date& d) const;
    bool operator<=(const Date& d) const;
    bool operator==(const Date& d) const;
    bool operator!=(const Date& d) const;
private:
    int _year;
    int _month;
    int _day;
};

// date.cpp
#include "Date.h"

Date::Date(int year, int month, int day) {
    if (year >= 1900
        && month > 0 && month < 13
        && day > 0 && day <= GetMonthDay(year, month)) {
        _year = year;
        _month = month;
        _day = day;
    } else {
        cout << "Input illegal,reset to default value" << endl;
        _year = 1900;
        _month = 1;
        _day = 1;
    }
}

ostream& operator<<(ostream& _cout, const Date& d) {
    _cout << d._year << "-" << d._month << "-" << d._day << endl;
    return _cout;
}

istream& operator>>(istream& _cin, Date& d) {
    int year;
    int month;
    int day;
    _cin >> year;
    _cin >> month;
    _cin >> day;
    // new(&d)Date(year, month, day);
    d = Date(year, month, day);
    return _cin;
}

int Date::operator-(const Date& d) const {
    Date maxdate(*this);
    Date mindate(d);
    int flag = 1;
    if (*this < d) {
        maxdate = d;
        mindate = *this;
        flag = -1;
    }
    int days = 0;
    while (mindate != maxdate) {
        ++mindate;
        ++days;
    }
    return days * flag;
}

// ++d 前置++
Date& Date::operator++() {
    *this += 1;
    return *this;
}

// d++ 后置++
Date Date::operator++(int) {
    Date ret(*this);
    *this += 1;
    return ret;
}

// --d 前置--
Date& Date::operator--() {
    *this -= 1;
    return *this;
}

// d-- 后置--
Date Date::operator--(int) {
    Date tmp(*this);
    *this -= 1;
    return tmp;
}

Date Date::operator+(int day) const {
    Date ret = *this;
    ret += day;
    return ret;
}

Date Date::operator-(int day) const {
    Date ret(*this);
    ret -= day;
    return ret;
}

Date& Date::operator+=(int day) {
    if (day < 0) {
        return *this -= -day;
    }
    _day += day;
    while (_day > GetMonthDay(_year, _month)) {
        _day -= GetMonthDay(_year, _month);
        _month++;
        if (_month == 13) {
            ++_year;
            _month = 1;
        }
    }
    return *this;
}

Date& Date::operator-=(int day) {
    if (day < 0) {
        return *this += -day;
    }
    _day -= day;
    while (_day <= 0) {
        --_month;
        if (_month == 0) {
            _month = 12;
            --_year;
        }
        _day += GetMonthDay(_year, _month);
    }
    return *this;
}

bool Date::operator>(const Date& d) const {
    if (_year > d._year) {
        return true;
    } else if (_year == d._year) {
        if (_month > d._month) {
            return true;
        } else if (_month == d._month) {
            if (_day > d._day) {
                return true;
            }
        }
    }
    return false;
}

bool Date::operator>=(const Date& d) const {
    return *this > d || *this == d;
}

bool Date::operator<(const Date& d) const {
    return !(*this >= d);
}

bool Date::operator<=(const Date& d) const {
    return !(*this > d);
}

bool Date::operator==(const Date& d) const {
    return _year == d._year && _month == d._month && _day == d._day;
}

bool Date::operator!=(const Date& d) const {
    return !(*this == d);
}

int main() {
    Date date;
    cin >> date;
    cout << date << endl;
    return 0;
}
```

