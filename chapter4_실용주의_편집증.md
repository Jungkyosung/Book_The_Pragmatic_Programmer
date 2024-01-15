"여러분은 완벽한 소프트웨어를 만들 수 없다."

실용주의 프로그래머는 남들을 믿지 않고 실수를 방지하기 위해 방어적 코딩을 하며, 본인 자신도 믿지 않고 완벽한 코드를 작성할 수 없음을 알기에 자신의 실수에 대비한 방어책을 마련한다.

---
#### Topic23. 계약에 의한 설계

순수한 함수처럼 어떤 인풋이 들어오면 어떤 아웃풋이 나오는지 그리고 그런 인풋의 조건과 아웃풋의 결과를 명확히 인지하여 설계하라.
테스트 주도 개발을 통해 이런 부분을 현실화할 수 있다. (하지만 TDD는 정상적인 경로를 위주로 테스트할 경우가 많다. 세상은 비정상적인 접근이 많다.)

---
#### Topic24. 죽은 프로그램은 거짓말을 하지 않는다.

일찍 작동을 멈춰라.

```
try do
  add_score_to_board(score);
rescue InvalidScore
  Logger.error("올바르지 않은 점수는 더할 수 없음. 종료합니다.");
  raise
rescue BoardServerDown
  Logger.error("점수를 더할 수 없음: 보드가 죽었음. 종료합니다.");
  raise
rescue StateTransaction
  Logger.error("점수를 더할 수 없음: 오래된 트랜잭션. 종료합니다.");
  raise
end
```

실용주의 프로그래머라면 다음과 같이 쓸 것이다.

```
add_score_to_board(score);
```
위와 같이 작성하는 이유
1. 애플리케이션 코드가 오류 처리 코드 사이에 묻히지 않음.
2. 코드의 결합도를 높이지 않음. 길게 쓴 코드에서는 해당 메서드가 발생시킬 수 있는 모든 예외를 다 나열해야 함. 예외가 추가되면 코드에 갱신이 생김.

가능한 한 빨리 문제를 발견하면 좀 더 일찍 시스템을 멈출 수 있으니 더 낫다.  
게다가 프로그램을 멈추는 것이 할 수 있는 최선인 경우가 흔하다.

---
#### Topic25. 단정적 프로그래밍

절대 일어나선 안되는 곳에 단정문(assert)을 활용해라. 단정문으로 불가능한 상황을 예방하라.

불변식을 조기에 확인하는 게 좋을 듯. 단정문까지는 모르겠고 객체 생성시에 불변식을 확인하는 작업을 하는 게 더 효율적인 것 같음.

---
#### Topic26. 리소스 사용의 균형

모든 것은 트레이드 오프를 고려해야 한다.

우리는 코딩할 때 언제나 리소스를 관리한다.  
리소스를 관리하는 하나의 원칙은 **'자신이 시작한 것은 자신이 끝내라'**이다.

나쁜 코드 예시)
```
def read_customer
  @customer_file = File.open(@name + ".rec", "r+")
  @balance       = BigDecimal(@customer_file.gets)
end

def write_customer
  @customer_file.rewind
  @customer_file.puts @balance.to_s
  @customer_file.close
end

def update_customer(transaction_amount)
  read_customer
  @balance += transaction_amount
  write_customer
end
```

왜 나쁜가? 명세가 바뀌었다는 이야기를 들은 불운한 유지 보수 프로그래머를 생각해 보자.  
잔액은 새로운 값이 음수가 아닌 경우에만 갱신되어야 한다는 조건이 추가되었다. 소스 코드를 찾아서 update_customer를 수정한다.

```
def update_customer(transaction_amount)
  read_customer
  new_balance = @balance + transaction_amount
  if (new_balance >= 0.00)
    @balance = new_balance
    write_customer
  end
end
```

이렇게 할 경우 배포 몇 시간 후 파일이 너무 많이 열려 있다는 오류와 함께 죽어 버린다.  
알고 보니 write_customer가 몇몇 상황에서는 호출되지 않았고, 그런 경우 파일이 닫히지 않았다.  

```
def update_customer(transaction_amount)
  read_customer
  new_balance = @balance + transaction_amount
  if (new_balance >= 0.00)
    @balance = new_balance
    write_customer
  else
    @customer_file.close # 나쁜 방법!
  end
end
```

올바르게 바꿔보자.

```
def read_customer(file)
  @balance = BigDecimal(file.gets)
end

def write_customer(file)
  file.rewind
  file.puts @balance.to_s
end

def update_customer(transaction_amount)
  file = File.open(@name + ".rec" + "r+"
  read_customer(file)
  @balance += transaction_amount
  write_customer(file)
  file.close
end
```

파일 객체를 인스턴스 변수에 저장하지 않고 매개 변수로 전달받도록 코드를 바꾸었다.  
이제 모든 책임은 update_customer로 옮겨졌다.  
열기와 닫기가 한 곳에 있게 됐다.

**중첩 할당**
리소스 할당의 기본 패턴을 확장해서 한 번에 여러 리소스를 사용하는 루틴에 적용할 수 있다.
- 리소스를 할당한 순서의 역순으로 해제하라.
- 코드의 여러 곳에서 동일한 구성의 리소스들을 할당하는 경우에는 언제나 같은 순서로 할당해야 교착 가능성을 줄일 수 있다.
    프로세스A가 resource1을 이미 확보하고서 resource2를 획득하려고 하고 있는데 프로세스 B는 resource2를 확보한 상태로 resource1을 막 요청하려고 한다면, 두 프로세스 모두 영원히 기다리게 될 것이다.

예외가 발생할 경우엔 finally절에서 메모리 해제를 해준다.


  
