# 고정소수점 연산
`fixed-point`라는 말은 참 재미있는 단어입니다. **고정소수점**을 의미하는 동시에 *고정점*도 의미하기 때문입니다. 이 부분에서는 고정소수점을 의미합니다.

아무튼, 운영체제에서는 여러 가지 문제가 있어서 부동소수점을 사용하지 않으니 정수 연산으로 소수를 표현 해야합니다. 이를 위해서 정수형을 쪼갤 필요가 있습니다. 당연히 부호는 잘 표기해야하고, 정수부와 소수부를 잘 나누어서 표현해봅시다.

```
31    30             13              0
+------+--------------+--------------+
| Sign | Integer Part | Decimal Part |
+------+--------------+--------------+
```
위와 같이 표기하면 최댓값은 131071.99993896484375, 최솟값은 -131071.99993896484375가 되겠습니다.

## 고정소수점으로의 변환, 역변환
고정소수점으로의 변환 과정은 쉽습니다. 나머지 비트가 전부 0이고, 정수부의 최하위 비트가 1인 수를 곱하면 됩니다. 즉, 정수에 1.0(2^14)을 곱하면 고정소수점으로 변환됩니다. 역변환 과정은 변환의 역연산입니다. 즉, 고정소수점을 1.0(2^14)으로 나누면 정수로 변환됩니다. 만약 131072가 넘는 값이 들어온다면 오버플로우로 인해 당연히 잘리게 되므로 주의해야합니다. 

## 고정소수점 연산
그 이후에 두 고정소수점 타입끼리의 연산은 단순히 두 수를 더하고, 빼면 됩니다. 이 때 곱하는 과정은 소수점을 곱하는데 문제가 있어서 오른쪽을 다시 정수로 만들고 곱합니다. 나눗셈도 동일합니다. 고정소수점과 정수에 대한 연산의 경우 곱하고 나누는 과정은 단순하게 처리할 수 있지만, 더하고 빼는 연산은 정수부만을 빼야하므로 정수를 고정소수점으로 바꾸고 연산을 수행합니다.

이를 정리하면 아래와 같습니다.
|설명|연산|결과|
|---|---|---|
|n을 고정소수점으로 변환|n * (1<<14)|고정소수점|
|x를 정수로 변환|x / (1<<14)|정수|
|x + y|x + y|고정소수점|
|x - y|x - y|고정소수점|
|x + n|x + n * (1<<14)|고정소수점|
|x - n|x - n * (1<<14)|고정소수점|
|x * y|(extended x) * y / (1<<14)|고정소수점|
|x * n|x * n|고정소수점|
|x / y|(extended x) * (1<<14) / y|고정소수점|
|x / n|x / n|고정소수점|
