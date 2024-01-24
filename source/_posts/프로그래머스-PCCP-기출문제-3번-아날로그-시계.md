---
title: 프로그래머스 PCCP 기출문제 3번 - 아날로그 시계 (JAVA)
date: 2024-01-23 17:33:57
tags:
---

# 문제

- 링크
https://school.programmers.co.kr/learn/courses/30/lessons/250135

시침, 분침, 초침이 있는 아날로그시계가 있습니다. 시계의 시침은 12시간마다, 분침은 60분마다, 초침은 60초마다 시계를 한 바퀴 돕니다. 따라서 시침, 분침, 초침이 움직이는 속도는 일정하며 각각 다릅니다. 이 시계에는 초침이 시침/분침과 겹칠 때마다 알람이 울리는 기능이 있습니다. 당신은 특정 시간 동안 알람이 울린 횟수를 알고 싶습니다.

다음은 0시 5분 30초부터 0시 7분 0초까지 알람이 울린 횟수를 세는 예시입니다.

{% asset_img image.png %}

가장 짧은 바늘이 시침, 중간 길이인 바늘이 분침, 가장 긴 바늘이 초침입니다.
알람이 울리는 횟수를 세기 시작한 시각은 0시 5분 30초입니다.
이후 0시 6분 0초까지 초침과 시침/분침이 겹치는 일은 없습니다.

{% asset_img image-1.png %}

약 0시 6분 0.501초에 초침과 시침이 겹칩니다. 이때 알람이 한 번 울립니다.
이후 0시 6분 6초까지 초침과 시침/분침이 겹치는 일은 없습니다.

{% asset_img image-2.png %}

약 0시 6분 6.102초에 초침과 분침이 겹칩니다. 이때 알람이 한 번 울립니다.
이후 0시 7분 0초까지 초침과 시침/분침이 겹치는 일은 없습니다.
0시 5분 30초부터 0시 7분 0초까지는 알람이 두 번 울립니다. 이후 약 0시 7분 0.584초에 초침과 시침이 겹쳐서 울리는 세 번째 알람은 횟수에 포함되지 않습니다.

<br>

# 초기 접근 방법과 시행착오

처음에는 초기 시침, 분침, 초침의 각도를 설정하고 초를 1초씩 증가하면서 변경되는 초침의 각도와 분침의 각도, 시침의 각도를 비교해서

`현재 초침의 각도 ≤ 현재 분침의 각도 ≤ 1초 뒤 분침의 각도 ≤ 1초 뒤 시침의 각도`

를 만족하면 케이스를 하나 증가시키는 방법으로 접근했다.

그래서
```java
public int solution(int h1, int m1, int s1, int h2, int m2, int s2) {
    // 원래 각도에 120을 곱하자
    int hourDegree = h1 % 12 * 30 * 120 + m1 * 60 + s1;

    int minDegree = m1 * 6 * 120 + s1 * 12;

    int secDegree = s1 * 6 * 120;

    int answer = 0;

    while (true) {
        if (secDegree <= minDegree && minDegree + 12 < secDegree + 720) {
            answer++;
        }

        if (secDegree <= hourDegree && hourDegree + 1 < secDegree + 720) {
            answer++;
        }

        if (secDegree == minDegree && minDegree == hourDegree) {
            answer--;
        }

        s1++;
        if (s1 == 60) {
            s1 = 0;
            m1++;
        }
        if (m1 == 60) {
            m1 = 0;
            h1++;
        }
        if (h1 == h2 && m1 == m2 && s1 == s2)
            break;
        secDegree = s1 * 6 * 120;
        minDegree = m1 * 6 * 120 + s1 * 12;
        hourDegree = h1 % 12 * 30 * 120 + m1 * 60 + s1;
    }

    return answer;
}
```


이렇게 접근했더니 11시 59분 30초부터 12시 0분 0초까지 돌아갈 때 답이 맞지 않았다.

<br>

# 문제 해결

도저히 풀기 어려운 와중에 수학적인 접근법을 제시해 본 사람이 있었다.

## 알고리즘

1. 0시 0분 0초를 기준으로 H시 M분 S초까지의 총 시간을 초 단위로 구한다
2. 해당 시간동안 초침이 시침과 만나는 횟수를 구한다
3. 해당 시간동안 초침이 분침과 만나는 횟수를 구한다
4. 위 (2)와 (3)을 더한다
5. 시침, 분침, 초침이 모두 겹치는 0시 0분 0초와 12시 0분 0초가 포함되면 1을 뺀다
6. (h2시 m2분 s2초까지 횟수) - (h1시 m1분 s1초까지 횟수) = 최종 정답
   
<br>

### 초침과 시침 만나는 횟수 구하기
<br>

0시 0분 0초에 모두 겹쳐있는 상태에서 출발한다고 가정해보자.

초침은 1초에 6도를 돈다.

시침은 1시간에 30도를 돌아가고 1시간은 3600초이므로 시침은 1초 동안 {% katex %} 30 / 3600 {% endkatex %}도 즉 {% katex %} 1 \over 120{% endkatex %} 도 돌아간다.

초침이 1바퀴를 돌아 제자리로 온 뒤에 다시 움직여서 시침과 만나는 시간을 식으로 세우면
{% katex %} 
6t - 360 = {1 \over 120}t
{% endkatex %} 

여기서 좌변은 t초 후의 초침의 각이고 우변은 t초 후의 시침의 각이다.

식을 정리하면,
{% katex %}
719t = 43200
{% endkatex %}

이므로 {% katex %} t = {43200 \over 719} {% endkatex %}초 후에 초침과 시침이 만난다는 것을 알 수 있다.

초침과 시침이 만난 후에는 다시 둘이 겹쳐있는 상태이므로 또 {% katex %}43200 \over 719{% endkatex %}초 후에 만나게 되므로, 초침과 시침이 만나는 주기는 {% katex %}{43200 \over 719}{% endkatex %}초가 된다.

그러면 t초 동안 만나는 횟수는 t를 {% katex %}43200 \over 719{% endkatex %}로 나누면 되는 것이다.

### 초침과 분침 만나는 횟수 구하기

분침은 1초에 {% katex %}1\over10{% endkatex %}도 씩 움직인다.

시침과 마찬가지로 식을 세우면

{% katex %}6t - 360 = {1\over10}t{% endkatex %}

이므로 정리하면

{% katex %}59t = 3600{% endkatex %}

이 되어서 초침과 분침은 {% katex %}3600\over59{% endkatex %}초마다 한번 씩 만나게 되는 걸 알 수 있다.

<br>

## 코드로 옮기기

총 시간을 구하는 함수와 해당 시간동안 분침, 시침과 만나는 횟수를 구하는 함수를 따로 분리하였다.

```java
/**
 * h시 m분 s초 까지 총 시간(초)
 */
int totalSec(int h, int m, int s) {
    return 3600 * h + 60 * m + s;
}

/**
 * 0시 0분 0초부터 h시 m분 s초 까지의 알람 횟수
 * 0시 0분 0초에 울리는 알람은 제외됨.
 */
int countAlarm(int h, int m, int s) {
    int totalSec = totalSec(h, m, s);

    // 초침과 분침은 3600/59초마다 만남
    int minAlarm = totalSec * 59 / 3600;

    // 초침과 시침은 43200/719초마다 만남
    int hourAlarm = totalSec * 719 / 43200;

    // 12시 0분 0초 이상인 경우에는 시침 분침 초침이 겹치므로 1회 빼야함
    int exception = totalSec >= 43200 ? 1 : 0;

    return minAlarm + hourAlarm - exception;
}
```

<br>

그리고 문제에서 시작 시간이 0시 0분 0초일 때 1회 알람이 울린다고 설명이 있으므로

알고리즘에서 세웠던 식에 추가적으로 작업이 필요했다.

### 최종 코드

```java
class Solution {
    public int solution(int h1, int m1, int s1, int h2, int m2, int s2) {
        // 시작 시간이 0시 0분 0초이거나 12시 0분 0초인 경우 시작하자마자 1회 울리므로 예외 처리
        int exception = (h1 % 12 == 0 && m1 == s1 && s1 == 0) ? 1 : 0;
        return countAlarm(h2, m2, s2) - countAlarm(h1, m1, s1) + exception;
    }

    /**
     * h시 m분 s초 까지 총 시간(초)
     */
    int totalSec(int h, int m, int s) {
        return 3600 * h + 60 * m + s;
    }

    /**
     * 0시 0분 0초부터 h시 m분 s초 까지의 알람 횟수
     * 0시 0분 0초에 울리는 알람은 제외됨.
     */
    int countAlarm(int h, int m, int s) {
        int totalSec = totalSec(h, m, s);

        // 초침과 분침은 3600/59초마다 만남
        int minAlarm = totalSec * 59 / 3600;

        // 초침과 시침은 43200/719초마다 만남
        int hourAlarm = totalSec * 719 / 43200;

        // 12시 0분 0초 이상인 경우에는 시침 분침 초침이 겹치므로 1회 빼야함
        int exception = totalSec >= 43200 ? 1 : 0;

        return minAlarm + hourAlarm - exception;
    }
}

```

# 느낀점

사실 수학적인 풀이는 다른 사람의 풀이를 약간 참고하였는데

알고리즘 문제 풀이를 할 때는 역시 프로그래밍적 구현도 중요하지만 수학적인 접근 방법을 먼저 생각해보는 것이 좋은 것 같다.
