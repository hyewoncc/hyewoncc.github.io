---
emoji: headers/how-can-collect-time.png
title: '새는 시간을 줄이고 싶다는 고민'
date: '2023-07-02 22:25:00'
author: 써머
tags: etc 회고
categories: etc 회고
---

지난 반 년간 회사에서 한 '개발'을 돌아봤을 때, 절반은 Node.js의 레거시를 스프링으로 옮기고, 그에 수반되는 작업에 해당되고, 
나머지 절반은 동시에 새로운 요구사항을 Node.js와 옮긴 신규 시스템에 붙이는 일에 해당되는 것 같다. '달리는 기차의 바퀴를 갈아 끼우'면서 동시에 열차 칸을 열심히 붙이고 있다.  

나는 레거시라는 단어의 절묘함을 좋아한다. 
레거시가 있어서 회사가 먹고 살았고, 나도 들어올 수 있었다는 점에서 '유산'이라는 의미가 딱 맞다. 
그리고 동시에 언제까지나 유산에 의지해서 살 수 없다는 면도 그렇다. 
그래서 레거시를 부정적인 뜻으로만 사용하는 걸 보면 조금 슬퍼진다.  

어쨌든 레거시 코드를 보며 회사 도메인 정책을 파악하는 일이, 좋은 사내 문서와 더 좋은 살아있는 레퍼런스인 팀원들이 있음에도 만만치 않았다. 
예외 상황은 너무 많았고 코드가 복잡해 쉽게 파악할 수 없었으며, 아무도 그렇게 돌아가고 있는 줄 모르던 부분을 발견하기도 했다.  

<br>

![두뇌 풀 가동](full.png)

<div style="text-align:center; font-style:italic; color:grey;">
  레거시 코드에서 정책을 파악하는 나
</div>

<br>

그러다 팀에서 인프런 [박우빈 님의 실용적인 테스트 가이드](https://www.inflearn.com/course/practical-testing-%EC%8B%A4%EC%9A%A9%EC%A0%81%EC%9D%B8-%ED%85%8C%EC%8A%A4%ED%8A%B8-%EA%B0%80%EC%9D%B4%EB%93%9C)를 듣고 얘기하는 단기 스터디를 하게 되었다.  
그 중에 '테스트는 문서니까 개발자스러운 용어 보다 사용자에 가깝게 쓰라'는 말이 있었다.  

이 얘기를 듣기 전까지 대략 이런 식으로 노드 테스트 코드를 짜고 있었다.  

```javascript
describe('getTransferDiscountInfo', function () {

    beforeEach(() => {
        service._useRepo.getLastEndedUseBeforeTimeByUserId = jest.fn()
        service.getTransferSpan = jest.fn()
    })

    test('마지막 이용이 없다면 예외를 던진다', async () => {
        service._useRepo.getLastEndedUseBeforeTimeByUserId.mockResolvedValue(Promise.resolve(null))

        const { applicable } = await service.getTransferDiscountInfo(1, moment('2023-05-17 14:00:00'))

        expect(applicable).toEqual(false)
    })

    test('마지막 이용으로부터 환승 가능 시간이 지났다면 예외를 던진다' , async () => {
        service._useRepo.getLastEndedUseBeforeTimeByUserId.mockResolvedValue(Promise.resolve({
                use_end_at:'2023-05-17 13:00:00',
                use_area_id: 199
            }))
        service.getTransferSpan.mockResolvedValue(Promise.resolve({ minutes: 30 }))

        const { applicable } = await service.getTransferDiscountInfo(1, moment('2023-05-17 13:30:01'))

        expect(applicable).toEqual(false)
    })

    test('마지막 이용으로부터 환승 가능 시작 시간과 종료 시간을 반환한다', async () => {
        service._useRepo.getLastEndedUseBeforeTimeByUserId.mockResolvedValue(Promise.resolve({
                use_end_at:'2023-05-17 13:00:00',
                use_area_id: 199
            }))
        service.getTransferSpan.mockResolvedValue(Promise.resolve({ minutes: 30 }))

        const { applicable, from, to } = await service.getTransferDiscountInfo(1, moment('2023-05-17 13:00:10'))

        expect(applicable).toEqual(true)
        expect(from.format('YYYY-MM-DD HH:mm:ss')).toEqual('2023-05-17 13:00:00')
        expect(to.format('YYYY-MM-DD HH:mm:ss')).toEqual('2023-05-17 13:30:00')
    })
})
```

<br>

나쁘진 않은 것 같은데, 문서로 쓰기에 충분하지 않다는 생각이 든다. 
대표적으로 개발자의 언어로 쓰여진 '예외를 던진다' 부분이 그렇다.  

이 테스트를 처음 보는 상황이라 가정하고 봤을 때, 우선 예외를 던진다더니 예외를 검증하고 있지 않아서 이 테스트가 제대로 짜인 테스트가 맞는지 의구심이 든다. 
더 나아가 입사 초의 나라면 마지막 이용이 없다는 게 처음 이용했다는 뜻이 맞나? 내 추측이 맞나? 고민하다 정책을 파악하고 싶었을 뿐인데 구체 레벨로 내려가서 또 사투를 벌였을 것 같다.  

그래서 요즘은 팀 스터디 후 액션 플랜으로 나온 '도메인 용어를 사용해 누구나 이해할 수 있는 설명을 쓴다'는 작성 규칙에 따라 이렇게 쓰려고 노력하고 있다.  

```javascript
describe('현재 시간 기준으로 환승 가능 여부와 가능한 시간을 조회한다.', function () {

    beforeEach(() => {
        service._useRepo.getLastEndedUseBeforeTimeByUserId = jest.fn()
        service.getTransferSpan = jest.fn()
    })

    test('유저의 첫 이용이라면 환승이 적용되지 않는다.', async () => {
        service._useRepo.getLastEndedUseBeforeTimeByUserId.mockResolvedValue(Promise.resolve(null))

        const { applicable } = await service.getTransferDiscountInfo(1, moment('2023-05-17 14:00:00'))

        expect(applicable).toEqual(false)
    })

    describe('이용한 적이 있다면, 마지막 이용의 종료 시간에 환승 가능 시간을 더한다.', function () {

        test('이미 그 시간이 지났다면 환승 불가능하다.', async () => {
            service._useRepo.getLastEndedUseBeforeTimeByUserId.mockResolvedValue(Promise.resolve({
                use_end_at:'2023-05-17 13:00:00',
                use_area_id: 199
            }))
            service.getTransferSpan.mockResolvedValue(Promise.resolve({ minutes: 30 }))

            const { applicable } = await service.getTransferDiscountInfo(1, moment('2023-05-17 13:30:01'))

            expect(applicable).toEqual(false)
        })

        test('그 시간이 지나지 않았다면 환승 가능하다. 환승 가능 시작 시간과 종료 시간을 계산한다.', async () => {
            service._useRepo.getLastEndedUseBeforeTimeByUserId.mockResolvedValue(Promise.resolve({
                use_end_at:'2023-05-17 13:00:00',
                use_area_id: 199
            }))
            service.getTransferSpan.mockResolvedValue(Promise.resolve({ minutes: 30 }))

            const { applicable, from, to } = await service.getTransferDiscountInfo(1, moment('2023-05-17 13:00:10'))

            expect(applicable).toEqual(true)
            expect(from.format('YYYY-MM-DD HH:mm:ss')).toEqual('2023-05-17 13:00:00')
            expect(to.format('YYYY-MM-DD HH:mm:ss')).toEqual('2023-05-17 13:30:00')
        })
    })
})
```

![테스트 실행 결과](result.png)  

코드는 그대로지만 정책 파악에 훨씬 도움되는 테스트가 되었다.  

<br>

회사에서 정말 재미있게 일하다 시계를 보면, 늘 왜 벌써 퇴근 시간이야?! 를 외치게 된다.  

![아무것도 안했는데](offwork.jpeg)  

초반에는 이게 너무 신기했고, 내가 일에 잘 몰입하고 있다는 증거 같아서 기뻤는데... 
이 얘기를 했을 때 팀원이 지금은 그 기분을 만끽해도 좋지만, 아마 조만간 하루를 효율적으로 쓰기에 대해 고민하게 될 거라 했었다. 
일에 적응하니 그 시기가 금방 온 것 같다.  

왜 벌써 퇴근 시간이야를 진지하게 고민 하다 보면, 저번에 읽었던 코드 또 읽느라 낭비한 시간, 
정책을 찾느라 문서를 뒤지거나 누군가에게 물어봤던 시간(이러면 심지어 두 명분의 시간이 된다), 요구 사항을 잘못 이해해 잘못 짠 코드를 고친 시간, 
오류가 났는데 대응 메뉴얼을 만들어 두지 않아서 매번 내가 직접 무언가 수정해야했던 시간... 등등이 떠오르는 것이다.  

기술적으로 어려운 부분이 있어서 공부를 했거나 애쓴 시간은 아쉽긴 하지만 뭐 아까운 마음은 들지 않는다. 
오히려 새로운 걸 배웠으니 뿌듯한 마음도 있다. 
그런데 기술 외적으로 헤매던 시간을 생각하면 좀 많이 아깝다.  

테스트 코드 설명을 잘 쓰는 게 그렇게까지 중요한 일인가? 이번만 보고 말 문서 같은데 굳이 고쳐둬야 할까? 같은 방심이 모여서 일의 효율성을 떨어트리는 것 같다.  
그리고 사실 이런 일은... 해 두면 언젠가 모두에게 좋다는 건 아는데, 늘 지금 하기엔 아까운 마음이 든다.  

이런 자잘하게 개선이 필요한 일들을 나서서 해주는 좋은 팀원이 있는데, 나도 좀 더 적극적으로 나서서 해봐야겠다.  

<br>
